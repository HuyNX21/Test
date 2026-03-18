Tạo Instance phía Proxy:
 
DIVC_ProxyManager* DIVC_ProxyManager::getInstance
	pthread_t thread = pthread_self(); --> Lấy thread ID của thread hiện tại
	pthread_once(&once_control,DIVC_ProxyManagerThreadKeyCreate); --> khởi tạo pthread_once để đoạn mã này chỉ chạy qua đúng 1 lần
		void DIVC_ProxyManagerThreadKeyCreate
			pthread_key_create(&, DIVC_ProxyManager::pthread_key_destructor); --> lưu dữ liệu riêng trên thread hiện tại (tách biệt hoàn toàn với thread khác) vào 1 key: pthread_key_thread_finishing và tự động giải phóng dữ liệu khi thread kết thúc thông qua IVC_ProxyManager::pthread_key_destructor
			DIVC_LogMessage::Init( False )
		mProxyManagerList = ThreadSafeSingleton<ProxyManagerList>::getInstance(); --> tạo instance của ProxyManagerList đúng 1 lần thông qua pthread_once
		ScopeLock lock( mProxyManagerList->mutex() ); --> Ở đây đang khóa mutex của ThreadSafeMap và tự unlock khi ra khỏi hàm
		PROXY_MANAGER_LIST_IT_T it = mProxyManagerList->reference().find( thread ); --> tìm it xem có it nào khớp với key thread trong std::map<pthread_t , DIVC_ProxyManager*> hay không
		PROXY_MANAGER_LIST_IT_T it_end = mProxyManagerList->reference().end();
		if(it != it_end)
			return (*it).second --> Nếu đã tồn tại key thread thì sẽ trả về value là instance của DIVC_ProxyManager
		else
			DIVC_ProxyManager* add_instance = new DIVC_ProxyManager;
			mProxyManagerList->reference().insert( std::make_pair( thread, add_instance ));
			add_instance->init_ProxyMgr()
			void DIVC_ProxyManager::init_ProxyMgr
				ScopeLock lock( mSocketErrorMutex )
				void DIVC_ProxyManager::init_Map --> insert mCallbackFuncArgTbl pair giữa index vs lời gọi hàm để có thể gọi callback 
				mInstanceIdAndObjectContainer = ThreadSafeSingleton<ProxyIdAndObjectContainer>::getInstance();
				mSyncIdAndObjectContainer = ThreadSafeSingleton<DIVC_SyncIdAndObjectContainer>::getInstance();
				mObservableExecutor = new ObservableExecutor();
					void ObservableExecutor::buildObservableClassTbl --> mObservableClassTbl ké thừa lại std::map --> quản lý các object thông qua key 
				mObserverIdAndObjectContainer = ThreadSafeSingleton<ObserverIdAndObjectContainer>::getInstance()
				mProxyRequestManager = new DIVC_ProxyRequestManager( mConnectTid )
				mEpollFd = epoll_create1(EPOLL_CLOEXEC)
				[for( sockid = 0; sockid < DIVD_SOCKID_MAX; sockid++ )]
					connectWithServer(sockid)
					void DIVC_ProxyManager::connectWithServer(DIVD_SOCKID sockid)
						const char* DIVC_PortNumberGetter::createThatServerSocketFile()
						mSocketFd[sockid] = socket( AF_UNIX, SOCK_STREAM, 0)
						fcntl( mSocketFd[sockid], F_SETFD, FD_CLOEXEC )
						connect(mSocketFd[sockid], (struct sockaddr*)&servAddr, sizeof(servAddr))
						epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mSocketFd[sockid], &event)
					mMessageReceiver[sockid] = new DIVC_MessageReceiver(mSocketFd[sockid], mProxyRequestManager, mConnectPid, mConnectTid);
						DIVC_MessageReceiver::DIVC_MessageReceiver(int sock_fd, DIVC_RequestManager* pRequestManager, pid_t peerSys_pid, pid_t peerSys_tid)
						mSockFd = sock_fd
						mEpollFd = epoll_create1(EPOLL_CLOEXEC)
						struct epoll_event event = {(EPOLLIN | EPOLLRDHUP), reinterpret_cast<void*>(mSockFd)}
						epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mSockFd, &event)
						mPeerSys_pid = peerSys_pid
						mPeerSys_tid = peerSys_tid
						mHealthCheckHandler = new DIVC_MessageHandlerBase;
						mRequestHandler = new DIVC_MessageHandlerBase;
						mCallBackHandler = new DIVC_MessageHandlerBase;
						mObserverNotifyHandler = new DIVC_MessageHandlerBase;
						mErrorHandler = new DIVC_MessageHandlerBase;
						mTimeOutHandler = new DIVC_MessageHandlerBase;
						mRequestManager = pRequestManager
					mMessageReceiver[sockid]->registHealthCheckHandler(new DIVC_ProxyHelthCheckMessageHandler(*this))
						DIVC_ProxyHelthCheckMessageHandler::DIVC_ProxyHelthCheckMessageHandler(DIVC_ProxyManager& proxyManager) : mProxyManager(proxyManager)
						/class DIVC_ProxyHelthCheckMessageHandler : public DIVC_MessageHandlerBase/
						void DIVC_MessageReceiver::registHealthCheckHandler(DIVC_MessageHandlerBase* handler)
							mHealthCheckHandler = handler
					mMessageReceiver[sockid]->registRequestHandler(new DIVC_ProxyRequestMessageHandler(*mProxyRequestManager, *this));
						mRequestHandler = handler
					mMessageReceiver[sockid]->registCallBackHandler(new DIVC_ProxyCallBackMessageHandler(*this));
						mCallBackHandler = handler
					mMessageReceiver[sockid]->registObserverNotifyHandler(new DIVC_ProxyObseverNotifyMessageHandler(*this));
						mObserverNotifyHandler = handler
					mMessageReceiver[sockid]->registErrorHandler(new DIVC_ProxyErrorMessageHandler(*this, sockid));
						mErrorHandler = handler
					mIsSocketConnected[sockid] = true
				[/for/]
				[for( sockid = 0; sockid < DIVD_SOCKID_MAX; sockid++ )]
					sendControlData(sockid)
					void DIVC_ProxyManager::sendControlData(DIVD_SOCKID sockid)
						data.sys_pid = getpid();
						data.sys_tid = (pid_t)syscall( SYS_gettid );
						pthread_getschedparam(pthread_self(),&policy,&param) --> lấy sched_priority của thread hiện tại
						data.priority = param.sched_priority;
						data.sockid = sockid;
						sendSocket( mSocketFd[sockid], (unsigned char*)&data,sizeof(data), sockid )
						int DIVC_ProxyManager::sendSocket(int sock_fd, unsigned char *buf, MESSAGE_SIZE_T size, DIVD_SOCKID sockid)
							struct epoll_event events[1]
							events[0].events = (EPOLLOUT | EPOLLRDHUP);
							events[0].data.fd = sock_fd;
							epoll_ctl(mEpollFd, EPOLL_CTL_MOD, sock_fd, &events[0])
							epoll_wait(mEpollFd, events, sizeof(events)/sizeof(events[0]), -1) --> block cho tới khi Socket sẵn sàng để ghi (EPOLLOUT) — tức là kernel send buffer có chỗ trống
							(int)send(sock_fd, buf + sendedsize, size - sendedsize, MSG_NOSIGNAL )
							events[0].events = 0
							events[0].data.fd = sock_fd
							epoll_ctl(mEpollFd, EPOLL_CTL_MOD, sock_fd, &events[0])
							--> Vòng lặp while để làm gì?
							--> Vì send() không đảm bảo gửi hết toàn bộ size bytes trong một lần, nên vòng lặp tiếp tục gửi phần còn lại (buf + sendedsize) cho đến khi sendedsize >= size thì mới break.
					mReceiveProcessor[sockid] = new ReceiveProcessor
					mReceiveProcessor[sockid]->setParam(this, sockid)
					void DIVC_ProxyManager::ReceiveProcessor::setParam(DIVC_ProxyManager *manager, DIVD_SOCKID sockid)
						mProxyManager = manager
						mSockId = sockid
					(DIVD_SOCKID_SYNC  == sockid) --> mReceiveThread[sockid] = new Thread(mReceiveProcessor[sockid], "ProxyRcvTh" , DEF_PROXY_RECV_CTRL_THREAD_PRIO_RCV)
					(DIVD_SOCKID_ASYNC == sockid) --> mReceiveThread[sockid] = new Thread(mReceiveProcessor[sockid], "ProxyAsyncRcvTh", DEF_PROXY_RECV_CTRL_THREAD_PRIO_ASYNCRCV)
					mReceiveThread[sockid]->setStackSize(DEF_STACK_SIZE_PROXYRCVTH)
					mReceiveThread[sockid]->run()
					void Thread::run()
						int error = pthread_attr_init(&mThreadAttribute); --> chuẩn bị attribute để set các thuộc tính khác
						error = pthread_attr_setstacksize(&mThreadAttribute, stackSize) --> Đặt kích thước stack cho thread
						error = pthread_attr_setdetachstate(&mThreadAttribute, detachstate) --> Đặt chế độ joinable hoặc detached cho thread
							PTHREAD_CREATE_JOINABLE → có thể join
							PTHREAD_CREATE_DETACHED → tự hủy, không join được
						error = pthread_attr_setscope( &mThreadAttribute , PTHREAD_SCOPE_SYSTEM ) --> Đặt competition scope (phạm vi lập lịch):
							PTHREAD_SCOPE_SYSTEM : Thread cạnh tranh trực tiếp với các thread khác trong hệ thống (thực tế Linux luôn sử dụng cái này).
							PTHREAD_SCOPE_PROCESS: Thread cạnh tranh trong phạm vi nội bộ process → Linux không hỗ trợ (set sẽ báo lỗi hoặc bị ignore).
						int error = pthread_create(&mThread, &mThreadAttribute, ThreadEntry, (void*)this) --> Tạo thread mới với các attribute đã set ở trên
							void* Thread::ThreadEntry(void *arg)
								thread->mRunnable->run()
									void DIVC_ProxyManager::ReceiveProcessor::run
										void DIVC_ProxyManager::Recv_ProxyMgr(DIVD_SOCKID sockid)
											...
						pthread_attr_destroy(&mThreadAttribute)
					mReceiveControl = DIVC_ReceiveControl::newInstance(mConnectPid,mSocketFd[sockid], this, true )
					DIVC_ReceiveControl*  DIVC_ReceiveControl::newInstance(pid_t peer_pid, int sfd, DIVC_ReceiveControlObserver* manager, bool bPoxy)
						ScopeLock lock( DIVC_ReceiveControl::mMutex )
						DIVT_CTRL_MAP_T::iterator it = mCtrlMap.find( peer_pid );
						DIVT_CTRL_MAP_T::iterator it_end = mCtrlMap.end();
						if(it == it_end)
							data.reference = 1;
							data.pReceiveControl = pReceiveControl = new DIVC_ReceiveControl;
							mCtrlMap.insert(  DIVT_CTRL_MAP_T::value_type( peer_pid, data ) )
							pReceiveControl->start(bPoxy,peer_pid)
							void DIVC_ReceiveControl::start(bool mode,pid_t pid)
								mProcessor = new Processor( this )
								mThread = new Thread(mProcessor, "ProxyRxCtrlTh", DEF_PROXY_RECV_CTRL_THREAD_PRIO_RX)
								mThread->run()
									...
									void* Thread::ThreadEntry(void *arg)
										thread->mRunnable->run()
											void DIVC_ReceiveControl::Processor::run()
												void DIVC_ReceiveControl::mainLoop()
													...
						else
							DIVS_CTRL_T& rData = (*it).second;
							rData.reference++;
							pReceiveControl = rData.pReceiveControl;
						FdInfo fdinfo;
						fdinfo.mManager = manager;
						fdinfo.m_FdState = DIVD_RXCTRL_STOP
						pReceiveControl->mFDMap.insert( DIVT_FD_MAP_T::value_type( sfd, fdinfo ) )
				[/for/]
