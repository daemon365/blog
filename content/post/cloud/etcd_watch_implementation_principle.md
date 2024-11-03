---
title: etcd watch 实现原理
date: '2024-06-10T14:16:00+08:00'
tags:
- boltdb
- etcd
- golang
- 数据库
- kubernetes
showToc: true
categories:
- cloud
---



## 介绍

在 etcd 中，watch 是一个非常重要的特性，它可以让客户端监控 etcd 中的 key 或者一组 key，当 key 发生变化时，etcd 会通知客户端。本文将介绍 etcd watch 的实现原理。


```BASH
etcdctl watch /test
# 当 /test 的值发生变化时，会输出如下信息
PUT
/test
a
PUT
/test
b
DELETE
/test
```

## watch 的 api

etcd watch api 是由 grpc stream 实现的，客户端通过 grpc stream 发送 watch 请求，etcd 会将 key 的变化通过 stream 返回给客户端。

```protobuf
rpc Watch(stream WatchRequest) returns (stream WatchResponse) {
      option (google.api.http) = {
        post: "/v3/watch"
        body: "*"
    };
}
```

## api 实现

```go
func (ws *watchServer) Watch(stream pb.Watch_WatchServer) (err error) {
	sws := serverWatchStream{
		lg: ws.lg,

		clusterID: ws.clusterID,
		memberID:  ws.memberID,

		maxRequestBytes: ws.maxRequestBytes,

		sg:        ws.sg,
		watchable: ws.watchable,
		ag:        ws.ag,

		gRPCStream:  stream,
		watchStream: ws.watchable.NewWatchStream(),
		// chan for sending control response like watcher created and canceled.
		ctrlStream: make(chan *pb.WatchResponse, ctrlStreamBufLen),

		progress: make(map[mvcc.WatchID]bool),
		prevKV:   make(map[mvcc.WatchID]bool),
		fragment: make(map[mvcc.WatchID]bool),

		closec: make(chan struct{}),
	}

	sws.wg.Add(1)
	go func() {
        // 开启一个 goroutine 处理新的 event 然后发送给客户端
		sws.sendLoop()
		sws.wg.Done()
	}()

	errc := make(chan error, 1)
	
	go func() {
        // 开启一个 goroutine 处理客户端发送的 watch 请求
		if rerr := sws.recvLoop(); rerr != nil {
			if isClientCtxErr(stream.Context().Err(), rerr) {
				sws.lg.Debug("failed to receive watch request from gRPC stream", zap.Error(rerr))
			} else {
				sws.lg.Warn("failed to receive watch request from gRPC stream", zap.Error(rerr))
				streamFailures.WithLabelValues("receive", "watch").Inc()
			}
			errc <- rerr
		}
	}()

	// 处理结束
	select {
	case err = <-errc:
		if err == context.Canceled {
			err = rpctypes.ErrGRPCWatchCanceled
		}
		close(sws.ctrlStream)
	case <-stream.Context().Done():
		err = stream.Context().Err()
		if err == context.Canceled {
			err = rpctypes.ErrGRPCWatchCanceled
		}
	}

	sws.close()
	return err
}

```

这里 主要的逻辑是开启两个 goroutine，一个用于处理客户端发送的 watch 请求，另一个用于处理新的 event 然后发送给客户端。

### sendLoop

```go
func (sws *serverWatchStream) sendLoop() {
	// watch ids that are currently active
	ids := make(map[mvcc.WatchID]struct{})
	// watch responses pending on a watch id creation message
	pending := make(map[mvcc.WatchID][]*pb.WatchResponse)

	interval := GetProgressReportInterval()
	progressTicker := time.NewTicker(interval)

	defer func() {
		progressTicker.Stop()
		// 清空chan ，清理待处理 event
		for ws := range sws.watchStream.Chan() {
			mvcc.ReportEventReceived(len(ws.Events))
		}
		for _, wrs := range pending {
			for _, ws := range wrs {
				mvcc.ReportEventReceived(len(ws.Events))
			}
		}
	}()

	for {
		select {
		case wresp, ok := <-sws.watchStream.Chan():
            // 从 watchStream.Chan() 中获取 event
            // 然后发送给客户端 
			if !ok {
				return
			}

			evs := wresp.Events
			events := make([]*mvccpb.Event, len(evs))
			sws.mu.RLock()
			needPrevKV := sws.prevKV[wresp.WatchID]
			sws.mu.RUnlock()
			for i := range evs {
				events[i] = &evs[i]
				if needPrevKV && !IsCreateEvent(evs[i]) {
					opt := mvcc.RangeOptions{Rev: evs[i].Kv.ModRevision - 1}
					r, err := sws.watchable.Range(context.TODO(), evs[i].Kv.Key, nil, opt)
					if err == nil && len(r.KVs) != 0 {
						events[i].PrevKv = &(r.KVs[0])
					}
				}
			}

			canceled := wresp.CompactRevision != 0
			wr := &pb.WatchResponse{
				Header:          sws.newResponseHeader(wresp.Revision),
				WatchId:         int64(wresp.WatchID),
				Events:          events,
				CompactRevision: wresp.CompactRevision,
				Canceled:        canceled,
			}

			// Progress notifications can have WatchID -1
			// if they announce on behalf of multiple watchers
			if wresp.WatchID != clientv3.InvalidWatchID {
				if _, okID := ids[wresp.WatchID]; !okID {
					// buffer if id not yet announced
					wrs := append(pending[wresp.WatchID], wr)
					pending[wresp.WatchID] = wrs
					continue
				}
			}

			mvcc.ReportEventReceived(len(evs))

			sws.mu.RLock()
			fragmented, ok := sws.fragment[wresp.WatchID]
			sws.mu.RUnlock()

			var serr error
			// gofail: var beforeSendWatchResponse struct{}
			if !fragmented && !ok {
				serr = sws.gRPCStream.Send(wr)
			} else {
				serr = sendFragments(wr, sws.maxRequestBytes, sws.gRPCStream.Send)
			}

			if serr != nil {
				if isClientCtxErr(sws.gRPCStream.Context().Err(), serr) {
					sws.lg.Debug("failed to send watch response to gRPC stream", zap.Error(serr))
				} else {
					sws.lg.Warn("failed to send watch response to gRPC stream", zap.Error(serr))
					streamFailures.WithLabelValues("send", "watch").Inc()
				}
				return
			}

			sws.mu.Lock()
			if len(evs) > 0 && sws.progress[wresp.WatchID] {
				// elide next progress update if sent a key update
				sws.progress[wresp.WatchID] = false
			}
			sws.mu.Unlock()

		case c, ok := <-sws.ctrlStream:
            // 处理客户端发送的 watch 请求
			if !ok {
				return
			}

			if err := sws.gRPCStream.Send(c); err != nil {
				if isClientCtxErr(sws.gRPCStream.Context().Err(), err) {
					sws.lg.Debug("failed to send watch control response to gRPC stream", zap.Error(err))
				} else {
					sws.lg.Warn("failed to send watch control response to gRPC stream", zap.Error(err))
					streamFailures.WithLabelValues("send", "watch").Inc()
				}
				return
			}

			// track id creation
			wid := mvcc.WatchID(c.WatchId)

			verify.Assert(!(c.Canceled && c.Created) || wid == clientv3.InvalidWatchID, "unexpected watchId: %d, wanted: %d, since both 'Canceled' and 'Created' are true", wid, clientv3.InvalidWatchID)

			if c.Canceled && wid != clientv3.InvalidWatchID {
				delete(ids, wid)
				continue
			}
			if c.Created {
				// flush buffered events
				ids[wid] = struct{}{}
				for _, v := range pending[wid] {
					mvcc.ReportEventReceived(len(v.Events))
					if err := sws.gRPCStream.Send(v); err != nil {
						if isClientCtxErr(sws.gRPCStream.Context().Err(), err) {
							sws.lg.Debug("failed to send pending watch response to gRPC stream", zap.Error(err))
						} else {
							sws.lg.Warn("failed to send pending watch response to gRPC stream", zap.Error(err))
							streamFailures.WithLabelValues("send", "watch").Inc()
						}
						return
					}
				}
				delete(pending, wid)
			}

		case <-progressTicker.C:
			sws.mu.Lock()
			for id, ok := range sws.progress {
				if ok {
					sws.watchStream.RequestProgress(id)
				}
				sws.progress[id] = true
			}
			sws.mu.Unlock()

		case <-sws.closec:
			return
		}
	}
}

```

这里使用了 for select 循环：
1. 从 watchStream.Chan() 中获取 event 然后发送给客户端。
2. 处理客户端发送的 watch 请求。
3. dispatch progress 事件。
4. 处理结束。


### recvLoop

```go
func (sws *serverWatchStream) recvLoop() error {
	for {
		req, err := sws.gRPCStream.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			return err
		}

		switch uv := req.RequestUnion.(type) {
		case *pb.WatchRequest_CreateRequest:
			if uv.CreateRequest == nil {
				break
			}

			creq := uv.CreateRequest
			if len(creq.Key) == 0 {
				// \x00 is the smallest key
				creq.Key = []byte{0}
			}
			if len(creq.RangeEnd) == 0 {
				// force nil since watchstream.Watch distinguishes
				// between nil and []byte{} for single key / >=
				creq.RangeEnd = nil
			}
			if len(creq.RangeEnd) == 1 && creq.RangeEnd[0] == 0 {
				// support  >= key queries
				creq.RangeEnd = []byte{}
			}

			err := sws.isWatchPermitted(creq)
			if err != nil {
				var cancelReason string
				switch err {
				case auth.ErrInvalidAuthToken:
					cancelReason = rpctypes.ErrGRPCInvalidAuthToken.Error()
				case auth.ErrAuthOldRevision:
					cancelReason = rpctypes.ErrGRPCAuthOldRevision.Error()
				case auth.ErrUserEmpty:
					cancelReason = rpctypes.ErrGRPCUserEmpty.Error()
				default:
					if err != auth.ErrPermissionDenied {
						sws.lg.Error("unexpected error code", zap.Error(err))
					}
					cancelReason = rpctypes.ErrGRPCPermissionDenied.Error()
				}

				wr := &pb.WatchResponse{
					Header:       sws.newResponseHeader(sws.watchStream.Rev()),
					WatchId:      clientv3.InvalidWatchID,
					Canceled:     true,
					Created:      true,
					CancelReason: cancelReason,
				}

				select {
				case sws.ctrlStream <- wr:
					continue
				case <-sws.closec:
					return nil
				}
			}

			filters := FiltersFromRequest(creq)

			wsrev := sws.watchStream.Rev()
			rev := creq.StartRevision
			if rev == 0 {
				rev = wsrev + 1
			}
			id, err := sws.watchStream.Watch(mvcc.WatchID(creq.WatchId), creq.Key, creq.RangeEnd, rev, filters...)
			if err == nil {
				sws.mu.Lock()
				if creq.ProgressNotify {
					sws.progress[id] = true
				}
				if creq.PrevKv {
					sws.prevKV[id] = true
				}
				if creq.Fragment {
					sws.fragment[id] = true
				}
				sws.mu.Unlock()
			} else {
				id = clientv3.InvalidWatchID
			}

			wr := &pb.WatchResponse{
				Header:   sws.newResponseHeader(wsrev),
				WatchId:  int64(id),
				Created:  true,
				Canceled: err != nil,
			}
			if err != nil {
				wr.CancelReason = err.Error()
			}
			select {
			case sws.ctrlStream <- wr:
			case <-sws.closec:
				return nil
			}

		case *pb.WatchRequest_CancelRequest:
			if uv.CancelRequest != nil {
				id := uv.CancelRequest.WatchId
				err := sws.watchStream.Cancel(mvcc.WatchID(id))
				if err == nil {
					sws.ctrlStream <- &pb.WatchResponse{
						Header:   sws.newResponseHeader(sws.watchStream.Rev()),
						WatchId:  id,
						Canceled: true,
					}
					sws.mu.Lock()
					delete(sws.progress, mvcc.WatchID(id))
					delete(sws.prevKV, mvcc.WatchID(id))
					delete(sws.fragment, mvcc.WatchID(id))
					sws.mu.Unlock()
				}
			}
		case *pb.WatchRequest_ProgressRequest:
			if uv.ProgressRequest != nil {
				sws.mu.Lock()
				sws.watchStream.RequestProgressAll()
				sws.mu.Unlock()
			}
		default:
			// we probably should not shutdown the entire stream when
			// receive an invalid command.
			// so just do nothing instead.
			sws.lg.Sugar().Infof("invalid watch request type %T received in gRPC stream", uv)
			continue
		}
	}
}
```


这里主要处理客户端发送的 watch 请求，然后发送给 ctrlStream。sendLoop 会从 ctrlStream 中获取 event 然后发送给客户端。


## WatchStream

这个 inferface 才是处理 watch 的主要逻辑


```go
// WatchStream 是一个接口，定义了一个流式处理watch请求的机制
type WatchStream interface {
	// Watch 创建一个观察者。观察者会监听在给定的键或范围 [key, end) 上发生的事件或已发生的事件。
	//
	// 整个事件历史都可以被观察到，除非被压缩。
	// 如果 "startRev" <= 0，watch 将观察在当前修订版本之后的事件。
	//
	// 返回的 "id" 是这个观察者的ID。它作为 WatchID 出现在通过 stream 通道发送到创建的观察者的事件中。
	// 当 WatchID 不等于 AutoWatchID 时，使用指定的 WatchID，否则返回自动生成的 WatchID。
	Watch(id WatchID, key, end []byte, startRev int64, fcs ...FilterFunc) (WatchID, error)

	// Chan 返回一个通道。所有的watch响应将被发送到这个返回的通道。
	Chan() <-chan WatchResponse

	// RequestProgress 请求给定ID的观察者的进度。响应只有在观察者当前同步时才会被发送。
	// 响应将通过与此流关联的 WatchResponse 通道发送，以确保正确的顺序。
	// 响应不包含事件。响应中的修订版本是观察者自同步以来的进度。
	RequestProgress(id WatchID)

	// RequestProgressAll 请求所有共享此流的观察者的进度通知。
	// 如果所有观察者都已同步，将向此流的任意观察者发送带有watch ID -1的进度通知，并返回 true。
	RequestProgressAll() bool

	// Cancel 通过给定ID取消观察者。如果观察者不存在，将返回错误。
	Cancel(id WatchID) error

	// Close 关闭通道并释放所有相关资源。
	Close()

	// Rev 返回流上观察到的KV的当前修订版本。
	Rev() int64
}

// WatchResponse 表示一个watch操作的响应。
type WatchResponse struct {
	// WatchID 是发送此响应的观察者的ID。
	WatchID WatchID

	// Events 包含所有需要发送的事件。
	Events []mvccpb.Event

	// Revision 是创建watch响应时KV的修订版本。
	// 对于正常响应，修订版本应该与Events中最后一个修改的修订版本相同。
	// 对于延迟响应的未同步观察者，修订版本大于Events中最后一个修改的修订版本。
	Revision int64

	// CompactRevision 在观察者由于压缩而被取消时设置。
	CompactRevision int64
}

// 实现了 WatchStream
// watchStream 包含共享一个流通道发送被观察事件和其他控制事件的观察者集合。
type watchStream struct {
	// 可观察对象（例如KV存储）
	watchable watchable
	// 用于发送watch响应的通道
	ch        chan WatchResponse

	// 互斥锁，保护以下字段
	mu sync.Mutex 
	// nextID 是为此流中下一个新观察者预分配的ID
	nextID   WatchID
	// 标志流是否已关闭
	closed   bool
	// 取消函数的映射，用于取消特定的观察者
	cancels  map[WatchID]cancelFunc
	// 观察者的映射，根据观察者ID索引
	watchers map[WatchID]*watcher
}

// Watch 在流中创建一个新的观察者并返回其 WatchID。
func (ws *watchStream) Watch(id WatchID, key, end []byte, startRev int64, fcs ...FilterFunc) (WatchID, error) {
	// 防止键 >= 结束键（按字典顺序）的错误范围
	// 带有 'WithFromKey' 的watch请求具有空字节范围结束
	if len(end) != 0 && bytes.Compare(key, end) != -1 {
		return -1, ErrEmptyWatcherRange
	}

	// 获取互斥锁
	ws.mu.Lock()
	defer ws.mu.Unlock()
	// 如果流已关闭，返回错误
	if ws.closed {
		return -1, ErrEmptyWatcherRange
	}

	// 自动生成 WatchID
	if id == clientv3.AutoWatchID {
		for ws.watchers[ws.nextID] != nil {
			ws.nextID++
		}
		id = ws.nextID
		ws.nextID++
	} else if _, ok := ws.watchers[id]; ok {
		return -1, ErrWatcherDuplicateID
	}

	// 创建新的观察者
	w, c := ws.watchable.watch(key, end, startRev, id, ws.ch, fcs...)

	// 保存取消函数和观察者
	ws.cancels[id] = c
	ws.watchers[id] = w
	return id, nil
}
// Chan 返回用于接收watch响应的通道。
func (ws *watchStream) Chan() <-chan WatchResponse {
	return ws.ch
}
// Cancel 取消具有给定ID的观察者。
func (ws *watchStream) Cancel(id WatchID) error {
	// 获取互斥锁
	ws.mu.Lock()
	cancel, ok := ws.cancels[id]
	w := ws.watchers[id]
	ok = ok && !ws.closed
	ws.mu.Unlock()

	// 如果观察者不存在或流已关闭，返回错误
	if !ok {
		return ErrWatcherNotExist
	}
	cancel()

	// 获取互斥锁
	ws.mu.Lock()
	// 在取消之前不删除观察者，以确保 Close() 调用时等待取消
	if ww := ws.watchers[id]; ww == w {
		delete(ws.cancels, id)
		delete(ws.watchers, id)
	}
	ws.mu.Unlock()

	return nil
}
// Close 关闭通道并释放所有相关资源。
func (ws *watchStream) Close() {
	// 获取互斥锁
	ws.mu.Lock()
	defer ws.mu.Unlock()

	// 取消所有观察者
	for _, cancel := range ws.cancels {
		cancel()
	}
	// 标记流已关闭并关闭通道
	ws.closed = true
	close(ws.ch)
	watchStreamGauge.Dec()
}
// Rev 返回流上观察到的KV的当前修订版本。
func (ws *watchStream) Rev() int64 {
	// 获取互斥锁
	ws.mu.Lock()
	defer ws.mu.Unlock()
	return ws.watchable.rev()
}
// RequestProgress 请求给定ID的观察者的进度。
func (ws *watchStream) RequestProgress(id WatchID) {
	// 获取互斥锁
	ws.mu.Lock()
	w, ok := ws.watchers[id]
	ws.mu.Unlock()
	// 如果观察者不存在，直接返回
	if !ok {
		return
	}
	// 请求进度
	ws.watchable.progress(w)
}
// RequestProgressAll 请求所有观察者的进度通知。
func (ws *watchStream) RequestProgressAll() bool {
	// 获取互斥锁
	ws.mu.Lock()
	defer ws.mu.Unlock()
	return ws.watchable.progressAll(ws.watchers)
}
```

1. Watch 方法：创建一个新的观察者，如果指定的范围不正确或观察者ID重复，则返回错误。否则，创建观察者并保存取消函数和观察者实例。
2. Chan 方法：返回用于接收watch响应的通道。
3. Cancel 方法：取消给定ID的观察者，删除相关的取消函数和观察者实例。
4. Close 方法：关闭所有观察者并释放资源。
5. Rev 方法：返回当前观察到的KV修订版本。
6. RequestProgress 方法：请求特定观察者的进度。
7. RequestProgressAll 方法：请求所有观察者的进度通知。

可以可到 当调用 Watch 的时候 每个 watchId 都会调用 `watchable.watch` 并把自己 ch 放入进去

## watchable


![](/images/ad802824-86e0-42a7-b84d-d2dfe2076620.png)

```go
// watchable 接口定义了可观察对象的行为
type watchable interface {
    // watch 创建一个新的观察者，用于监听指定键或范围[startRev, end)上的事件。
    // 返回观察者指针和取消函数。
    watch(key, end []byte, startRev int64, id WatchID, ch chan<- WatchResponse, fcs ...FilterFunc) (*watcher, cancelFunc)
    
    // progress 通知特定观察者当前的进度。
    progress(w *watcher)
    
    // progressAll 通知所有观察者当前的进度。
    // 如果所有观察者都已同步，则返回 true。
    progressAll(watchers map[WatchID]*watcher) bool
    
    // rev 返回当前观察到的修订版本。
    rev() int64
}


// watchableStore 是一个实现了 watchable 接口的结构体，代表一个可观察的存储
type watchableStore struct {
    // store 是一个指向基础存储的指针
    *store

    // mu 保护观察者组和批次。为了避免死锁，在锁定 store.mu 之前不应锁定 mu。
    mu sync.RWMutex

    // victims 是在 watch 通道上被阻塞的观察者批次
    victims []watcherBatch
    victimc chan struct{}

    // unsynced 包含所有需要同步已经发生的事件的未同步观察者
    unsynced watcherGroup

    // synced 包含所有与存储进度同步的观察者
    // 映射的键是观察者监听的键
    synced watcherGroup

    // stopc 是一个用于停止操作的通道
    stopc chan struct{}

    // wg 用于等待所有 goroutine 完成
    wg sync.WaitGroup
}

func (s *watchableStore) watch(key, end []byte, startRev int64, id WatchID, ch chan<- WatchResponse, fcs ...FilterFunc) (*watcher, cancelFunc) {
	// 创建一个新的观察者
	wa := &watcher{
		key:    key,
		end:    end,
		minRev: startRev,
		id:     id,
		ch:     ch,
		fcs:    fcs,
	}

	// 锁定 watchableStore 的互斥锁
	s.mu.Lock()
	// 锁定 store 的读写锁用于获取当前修订版本
	s.revMu.RLock()
	// 判断观察者是否与当前存储修订版本同步
	synced := startRev > s.store.currentRev || startRev == 0
	if synced {
		// 如果同步，设置最小修订版本为当前修订版本的下一个版本
		wa.minRev = s.store.currentRev + 1
		if startRev > wa.minRev {
			wa.minRev = startRev
		}
		// 将观察者添加到同步观察者组中
		s.synced.add(wa)
	} else {
		// 如果未同步，增加慢速观察者计数器
		slowWatcherGauge.Inc()
		// 将观察者添加到未同步观察者组中
		s.unsynced.add(wa)
	}
	// 解锁 store 的读写锁
	s.revMu.RUnlock()
	// 解锁 watchableStore 的互斥锁
	s.mu.Unlock()

	// 增加观察者计数器
	watcherGauge.Inc()

	// 返回观察者和取消函数
	return wa, func() { s.cancelWatcher(wa) }
}
```

### newWatchableStore

```go
func newWatchableStore(lg *zap.Logger, b backend.Backend, le lease.Lessor, cfg StoreConfig) *watchableStore {
	if lg == nil {
		lg = zap.NewNop()
	}
	s := &watchableStore{
		store:    NewStore(lg, b, le, cfg),
		victimc:  make(chan struct{}, 1),
		unsynced: newWatcherGroup(),
		synced:   newWatcherGroup(),
		stopc:    make(chan struct{}),
	}
	s.store.ReadView = &readView{s}
	s.store.WriteView = &writeView{s}
	if s.le != nil {
		// use this store as the deleter so revokes trigger watch events
		s.le.SetRangeDeleter(func() lease.TxnDelete { return s.Write(traceutil.TODO()) })
	}
	s.wg.Add(2)
	go s.syncWatchersLoop()
	go s.syncVictimsLoop()
	return s
}
```

### syncWatchersLoop

```go
// syncWatchersLoop 每100毫秒同步一次unsynced集合中的观察者。
func (s *watchableStore) syncWatchersLoop() {
	defer s.wg.Done()

	// 设置等待时间为100毫秒
	waitDuration := 100 * time.Millisecond
	delayTicker := time.NewTicker(waitDuration)
	defer delayTicker.Stop()

	for {
		// 锁定以获取未同步观察者的数量
		s.mu.RLock()
		st := time.Now()
		lastUnsyncedWatchers := s.unsynced.size()
		s.mu.RUnlock()

		unsyncedWatchers := 0
		// 如果有未同步观察者，同步这些观察者
		if lastUnsyncedWatchers > 0 {
			unsyncedWatchers = s.syncWatchers()
		}
		syncDuration := time.Since(st)

		// 重置定时器
		delayTicker.Reset(waitDuration)
		// 检查是否有更多待处理的工作
		if unsyncedWatchers != 0 && lastUnsyncedWatchers > unsyncedWatchers {
			// 公平对待其他存储操作，通过延长时间来避免占用太多资源
			delayTicker.Reset(syncDuration)
		}

		// 等待定时器或停止信号
		select {
		case <-delayTicker.C:
		case <-s.stopc:
			return
		}
	}
}

// syncWatchers 通过以下步骤同步未同步的观察者：
//  1. 从未同步观察者组中选择一组观察者
//  2. 迭代该组以获取最小修订版本并移除压缩的观察者
//  3. 使用最小修订版本获取所有键值对，并将这些事件发送给观察者
//  4. 从未同步组中移除已同步的观察者，并移动到同步组中
func (s *watchableStore) syncWatchers() int {
	// 锁定
	s.mu.Lock()
	defer s.mu.Unlock()

	// 如果没有未同步观察者，返回0
	if s.unsynced.size() == 0 {
		return 0
	}

	// 锁定存储的读写锁
	s.store.revMu.RLock()
	defer s.store.revMu.RUnlock()

	// 为了从未同步观察者中找到键值对，我们需要找到最小修订版本
	curRev := s.store.currentRev
	compactionRev := s.store.compactMainRev

	// 选择一组观察者
	wg, minRev := s.unsynced.choose(maxWatchersPerSync, curRev, compactionRev)
	minBytes, maxBytes := NewRevBytes(), NewRevBytes()
	minBytes = RevToBytes(Revision{Main: minRev}, minBytes)
	maxBytes = RevToBytes(Revision{Main: curRev + 1}, maxBytes)

	// UnsafeRange 返回键和值。在boltdb中，键是修订版本，值是实际的键值对。
	tx := s.store.b.ReadTx()
	tx.RLock()
	revs, vs := tx.UnsafeRange(schema.Key, minBytes, maxBytes, 0)
	evs := kvsToEvents(s.store.lg, wg, revs, vs)
	// 必须在kvsToEvents之后解锁，因为vs（来自boltdb内存）不是深拷贝。
	// 我们只能在Unmarshal之后解锁，这将进行深拷贝。
	// 否则我们将在boltdb重新mmap期间触发SIGSEGV。
	tx.RUnlock()

	// 创建一个新的观察者批次
	victims := make(watcherBatch)
	wb := newWatcherBatch(wg, evs)
	for w := range wg.watchers {
		if w.minRev < compactionRev {
			// 跳过因压缩而无法发送响应的观察者
			continue
		}
		w.minRev = curRev + 1

		eb, ok := wb[w]
		if !ok {
			// 将未通知的观察者移至同步
			s.synced.add(w)
			s.unsynced.delete(w)
			continue
		}

		if eb.moreRev != 0 {
			w.minRev = eb.moreRev
		}

		// 发送响应
		if w.send(WatchResponse{WatchID: w.id, Events: eb.evs, Revision: curRev}) {
			pendingEventsGauge.Add(float64(len(eb.evs)))
		} else {
			w.victim = true
		}

		// 处理受害者观察者
		if w.victim {
			victims[w] = eb
		} else {
			if eb.moreRev != 0 {
				// 保持未同步状态；还有更多要读取
				continue
			}
			s.synced.add(w)
		}
		s.unsynced.delete(w)
	}
	s.addVictim(victims)

	// 更新慢速观察者计数器
	vsz := 0
	for _, v := range s.victims {
		vsz += len(v)
	}
	slowWatcherGauge.Set(float64(s.unsynced.size() + vsz))

	return s.unsynced.size()
}
```


#### watcher & send


```go
type watcher struct {
	// the watcher key
	key []byte
	// end indicates the end of the range to watch.
	// If end is set, the watcher is on a range.
	end []byte

	// victim is set when ch is blocked and undergoing victim processing
	victim bool

	// compacted is set when the watcher is removed because of compaction
	compacted bool

	// restore is true when the watcher is being restored from leader snapshot
	// which means that this watcher has just been moved from "synced" to "unsynced"
	// watcher group, possibly with a future revision when it was first added
	// to the synced watcher
	// "unsynced" watcher revision must always be <= current revision,
	// except when the watcher were to be moved from "synced" watcher group
	restore bool

	// minRev is the minimum revision update the watcher will accept
	minRev int64
	id     WatchID

	fcs []FilterFunc
	// a chan to send out the watch response.
	// The chan might be shared with other watchers.
	ch chan<- WatchResponse
}

func (w *watcher) send(wr WatchResponse) bool {
	progressEvent := len(wr.Events) == 0

	if len(w.fcs) != 0 {
		ne := make([]mvccpb.Event, 0, len(wr.Events))
		for i := range wr.Events {
			filtered := false
			for _, filter := range w.fcs {
				if filter(wr.Events[i]) {
					filtered = true
					break
				}
			}
			if !filtered {
				ne = append(ne, wr.Events[i])
			}
		}
		wr.Events = ne
	}

	// if all events are filtered out, we should send nothing.
	if !progressEvent && len(wr.Events) == 0 {
		return true
	}
	select {
	case w.ch <- wr:
		return true
	default:
		return false
	}
}

```

### syncVictimsLoop

```go
// syncVictimsLoop 尝试将预先计算的观察者响应写入被阻塞的观察者通道
func (s *watchableStore) syncVictimsLoop() {
	defer s.wg.Done()

	for {
		// 尝试更新所有受害者观察者
		for s.moveVictims() != 0 {
			// 持续更新，直到所有受害者观察者都处理完毕
		}

		// 检查是否有受害者观察者
		s.mu.RLock()
		isEmpty := len(s.victims) == 0
		s.mu.RUnlock()

		var tickc <-chan time.Time
		if !isEmpty {
			tickc = time.After(10 * time.Millisecond)
		}

		// 等待10毫秒或收到新的受害者通知或停止信号
		select {
		case <-tickc:
		case <-s.victimc:
		case <-s.stopc:
			return
		}
	}
}

// moveVictims 尝试使用已存在的事件数据更新观察者
func (s *watchableStore) moveVictims() (moved int) {
	s.mu.Lock()
	victims := s.victims
	s.victims = nil
	s.mu.Unlock()

	var newVictim watcherBatch
	for _, wb := range victims {
		// 再次尝试发送响应
		for w, eb := range wb {
			// 观察者已观察到存储，直到但不包括 w.minRev
			rev := w.minRev - 1
			if w.send(WatchResponse{WatchID: w.id, Events: eb.evs, Revision: rev}) {
				pendingEventsGauge.Add(float64(len(eb.evs)))
			} else {
				if newVictim == nil {
					newVictim = make(watcherBatch)
				}
				newVictim[w] = eb
				continue
			}
			moved++
		}

		// 将完成的受害者观察者分配到未同步/同步组
		s.mu.Lock()
		s.store.revMu.RLock()
		curRev := s.store.currentRev
		for w, eb := range wb {
			if newVictim != nil && newVictim[w] != nil {
				// 无法发送watch响应，仍然是受害者
				continue
			}
			w.victim = false
			if eb.moreRev != 0 {
				w.minRev = eb.moreRev
			}
			if w.minRev <= curRev {
				s.unsynced.add(w)
			} else {
				slowWatcherGauge.Dec()
				s.synced.add(w)
			}
		}
		s.store.revMu.RUnlock()
		s.mu.Unlock()
	}

	// 如果仍然有未处理的受害者，重新添加到受害者列表中
	if len(newVictim) > 0 {
		s.mu.Lock()
		s.victims = append(s.victims, newVictim)
		s.mu.Unlock()
	}

	return moved
}
```
## Reference

- https://blog.csdn.net/qq_24433609/article/details/120653747
