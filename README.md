Buffer được ghi ở đâu, và Proxy biết khi nào sẵn sàng bằng cách nào
Kiến trúc thực tế dùng 2 thread + synchronized request/response qua sequence ID, không phải Stub "với tay" ghi vào buffer Proxy.

Hai thread phía Proxy
Caller thread — gọi DIVD_ProxyNewinstance_Singleton, chạy Send_ProxyMgr rồi block đợi.
Receive thread — ReceiveProcessor::run() (DIVC_ProxyManager.cpp:1595) chạy vòng lặp Recv_ProxyMgr (DIVC_ProxyManager.cpp:909-958) liên tục wait() trên socket → khi có data thì gọi messageReceive() đọc từ socket vào memory.
Luồng đầy đủ

Caller thread                              Receive thread (đã chạy sẵn)
─────────────                              ──────────────
[1605] marshalParams → sendbuf
[1608] Send_ProxyMgr(sendbuf)
        └─ sendRequestMessage
            └─ sendSynchronizedRequest
                ├─ gắn sequence_id vào request
                ├─ đăng ký "tôi đợi seqId này,
                │   đây là buffer của tôi (sendbuf)"
                ├─ ghi socket gửi đi  ─────IPC────▶  (Stub xử lý, gửi response
                │                                     kèm cùng sequence_id)
                └─ BLOCK trên condition var          │
                   /semaphore                        │
                                                     │
                                          [919] wait() trả về (socket có data)
                                          [938] messageReceive():
                                                ├─ đọc bytes từ socket
                                                ├─ parse header → lấy sequence_id
                                                ├─ tra bảng "ai đang đợi seqId này?"
                                                ├─ tìm ra caller thread + sendbuf
                                                ├─ MEMCPY response vào sendbuf
                                                └─ signal condition var
        ◀── unblock ───────────────────────────────┘
[1609] pt = sendbuf  (giờ đã chứa response)
[1612] unmarshalParams ĐỌC sendbuf, gán vào InstanceID, paraN
Trả lời chính xác 2 câu hỏi của bạn
1. Buffer sendbuf được "ghi đè" ở đâu?
Không phải Stub ghi trực tiếp. Mà là Receive thread của Proxy (trong cùng process với Proxy) — nó đọc bytes từ socket rồi memcpy vào sendbuf (buffer mà caller thread đã đăng ký kèm sequence_id). Stub chỉ "ghi" vào socket/IPC channel; phía nhận của Proxy mới ghi vào buffer.

2. Caller làm sao biết buffer đã sẵn sàng?
Bằng synchronization primitive (condition variable / semaphore / event), không phải polling. Cụ thể:

Trước khi gửi, sendSynchronizedRequest đăng ký (sequence_id → buffer + waiter) vào một bảng và block trên CV.
Receive thread khi nhận response, match sequence_id để tìm đúng caller, copy data, rồi signal CV.
Caller thread wake up → biết chắc buffer đã đầy đủ data → an toàn để unmarshalParams.
Đây là pattern synchronous RPC over async socket: socket bản chất bất đồng bộ và dùng chung cho nhiều request đồng thời, nên cần sequence_id để demux response về đúng caller, và CV để biến nó thành blocking call từ góc nhìn caller.
=======================================================
Step "MEMCPY response vào sendbuf" — chi tiết
Mọi thứ xảy ra trong DIVC_SynchronizedMediator. Có 2 thời điểm quan trọng:

A. Trước khi gửi — đăng ký buffer
Tại DIVC_SynchronizedMediator.cpp:235, trong sendSynchronizedRequest:


// �����f�[�^�����邽�߁A�A�h���X���o���Ă���
mSendBuffer = buffer;     // ← LƯU địa chỉ sendbuf của caller vào member
buffer ở đây chính là sendbuf mà caller (Proxy) đã truyền xuống từ Send_ProxyMgr(sendbuf, size). Mediator lưu lại con trỏ này như "địa chỉ ghi response".

Sau đó caller block tại dòng 312: mThreadConditionWait->wait();

B. Khi response về — copy vào đúng buffer đó
Receive thread cuối cùng gọi RecvReply() (DIVC_SynchronizedMediator.cpp:362). Tại dòng 380:


// ��ʂ���n���ꂽ�v���f�[�^�o�b�t�@�ɉ����f�[�^���R�s�[
memcpy( mSendBuffer, buffer, length );
//      ↑           ↑
//   địa chỉ     buffer tạm chứa response
//   sendbuf     vừa đọc từ socket
//   của caller
Đây chính là step "MEMCPY response vào sendbuf" tôi nói lúc trước. Cụ thể:

mSendBuffer = con trỏ tới sendbuf của caller (đã lưu ở bước A).
buffer (param thứ 2) = vùng nhớ tạm mà receive thread vừa read() từ socket vào.
length = size của response.
Sau memcpy xong, receive thread gọi mThreadConditionWait->wakeup() (dòng 396) → caller thread unblock → Send_ProxyMgr return → quay về DIVC_ProxyMethodTemplate.h:1612 chạy unmarshalParams đọc dữ liệu mới từ sendbuf.

Lưu ý nhỏ về macro
Đoạn này có ifdef:

#ifndef DEF_DISABLE_DIVLIB_DEL_RAM_THREAD: không memcpy ở đây — comment ghi "buffer đã được copy sẵn ở tầng trên" (variant tối ưu RAM, copy ở chỗ khác).
#else: có memcpy như trên — đây là code path mặc định bạn nên hình dung.
Cả 2 nhánh đều dẫn tới cùng một kết quả: sendbuf của caller chứa response trước khi wakeup().

Tổng kết chuỗi mắt xích

Proxy caller:  Send_ProxyMgr(sendbuf)
   └─ sendRequestMessage
      └─ ProxyRequestManager::sendSynchronizedRequest
         └─ SynchronizedMediator::sendSynchronizedRequest(buffer=sendbuf)
            ├─ mSendBuffer = buffer;          ← (A) lưu địa chỉ
            ├─ sendRequestData (gửi qua socket)
            └─ mThreadConditionWait->wait();  ← block

Receive thread: messageReceive → ... → 
   SynchronizedMediator::RecvReply(seqId, tmpBuf, len)
      ├─ memcpy(mSendBuffer, tmpBuf, len);    ← (B) GHI vào sendbuf
      └─ mThreadConditionWait->wakeup();      ← đánh thức caller
Vậy cụ thể: memcpy nằm tại DIVC_SynchronizedMediator.cpp:380, copy từ buffer tạm (đọc từ socket) sang đúng sendbuf mà Proxy đã đăng ký.
