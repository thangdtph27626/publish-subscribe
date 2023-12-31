# Hướng dẫn sử dụng cơ chế publish and subscribe để làm app chat trực tuyến


## 1. Giới thiệu về cơ chế Pub/Sub

Trong hệ thống phân tán lớn, message broker là một thành phần vô cùng quan trọng. Lý do vì sao mình xin thứ lỗi không giải thích ở đây. Hiện nay, có 2 messaging model phổ biến là message queuing và pub/sub messaging.

Message queuing được thiết kế để thực hiện danh sách task một cách lần lượt. Khi một message đến, hàng đợi sẽ tiếp nhận. Nếu task trước đó đã thực thi xong, message đó sẽ bị xóa, message kế tiếp mới được gửi đi để xử lý task tương ứng. Vậy thì ở đây, với message queuing, nếu như một message bị lỗi gì đó, thì có thế các message sau sẽ không được xử lý theo tuần tự nữa.

![image](https://github.com/thangdtph27626/publish-subscribe/assets/109157942/1c5be50f-bc9a-48c0-b13d-b0dd0c3c288e)

Pub/Sub messaging cũng làm nhiệm vụ giống như message queuing. Tuy nhiên, cơ chế của pub/sub có khác biệt:

- Pub - Publish: producer sẽ đẩy data vào một chanel.
- Sub - Subscribe: consumer đăng ký nhận dữ liệu từ một chanel.

 ![image](https://github.com/thangdtph27626/publish-subscribe/assets/109157942/0edb9b28-0e80-42d7-b364-47d7ee9beafc)

 Cái này giống như bạn xem Youtube. Bạn subscribe một channel. Khi một producer làm một video đẩy lên Youtube. Bạn sẽ nhận dữ liệu, tức xem video đó tại channel mà bạn đã subscribe. Khi bạn bè của bạn cũng subscribe channel đó, thì khi producer đẩy video lên, bạn và bạn bè của bạn cùng nhận được video để xem. Và tất nhiên, bạn có thể đăng ký nhiều channel.

Và tất nhiên mỗi messaging model có một ưu điểm riêng và tùy vào bài toán mà sử dụng chúng.

## 2. Mô hình hoạt động của Pub/Sub:


- Publisher tạo một chủ đề (topic) và gửi tin nhắn (message) đến chủ đề đó.
- Message sau khi publish sẽ được lưu trữ lại cho đến khi nhận được gói tin xác nhận (Acknowledgement) từ một subscriber bất kỳ đã tiêu thụ tin nhắn.
- Pub/Sub chuyển tiếp tin nhắn đến tất cả các subscription của chủ đề.
- Subscriber có thể nhận được tin nhắn từ Pub/Sub thông qua phương thức Pull hoặc Push.
- Subscriber gửi một gói tin xác nhận đến Pub/Sub cho mỗi tin nhắn nhận được.
- Dịch vụ Pub/Sub xóa các tin nhắn đã xác nhận khỏi hàng đợi tin nhắn của subscription.
![image](https://github.com/thangdtph27626/publish-subscribe/assets/109157942/b580fa44-9965-4684-a239-675b3e8153b2)

## 3. Demo bằng Python và Redis lib


### Setup
Install Redis: https://redis.io/download.
Install Python 3: https://www.python.org/downloads/.
Tạo project và môi trường:

```
$ mkdir pub_sub_messaging 
$ cd pub_sub_messaging
$ virtualenv venv
Using base prefix '/home/ha.hao.minh/.pyenv/versions/3.6.2'
New python executable in /home/ha.hao.minh/PycharmProjects/pub_sub_messaging/venv/bin/python3.6
Also creating executable in /home/ha.hao.minh/PycharmProjects/pub_sub_messaging/venv/bin/python
Installing setuptools, pip, wheel...done.
$ source venv/bin/activate
(venv)$ pip install redis
Collecting redis
  Using cached https://files.pythonhosted.org/packages/f5/00/5253aff5e747faf10d8ceb35fb5569b848cde2fdc13685d42fcf63118bbc/redis-3.0.1-py2.py3-none-any.whl
Installing collected packages: redis
Successfully installed redis-3.0.1

```

### Create Sub
Việc đầu tiên, consumer sẽ subcribe một channel. Sau khi cài Redis lên, bạn cần check lại xem service redis đã lên hay chưa:

```
$ sudo service redis status
● redis.service - Redis In-Memory Data Store
   Loaded: loaded (/etc/systemd/system/redis.service; disabled; vendor preset: enabled)
   Active: active (running) since T4 2018-12-19 07:41:43 WIB; 7s ago
 Main PID: 9830 (redis-server)
    Tasks: 4
   Memory: 11.6M
      CPU: 42ms
   CGroup: /system.slice/redis.service
           └─9830 /usr/local/bin/redis-server 0.0.0.0:6379         

Th12 19 07:41:43 framgia0185-pc redis-server[9830]:   `-._    `-._`-.__.-'_.-'    _.-'
Th12 19 07:41:43 framgia0185-pc redis-server[9830]:       `-._    `-.__.-'    _.-'
Th12 19 07:41:43 framgia0185-pc redis-server[9830]:           `-._        _.-'
Th12 19 07:41:43 framgia0185-pc redis-server[9830]:               `-.__.-'
Th12 19 07:41:43 framgia0185-pc redis-server[9830]: 9830:M 19 Dec 07:41:43.483 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the
Th12 19 07:41:43 framgia0185-pc redis-server[9830]: 9830:M 19 Dec 07:41:43.483 # Server initialized
Th12 19 07:41:43 framgia0185-pc redis-server[9830]: 9830:M 19 Dec 07:41:43.483 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue
Th12 19 07:41:43 framgia0185-pc redis-server[9830]: 9830:M 19 Dec 07:41:43.483 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and me
Th12 19 07:41:43 framgia0185-pc redis-server[9830]: 9830:M 19 Dec 07:41:43.520 * DB loaded from disk: 0.037 seconds
Th12 19 07:41:43 framgia0185-pc redis-server[9830]: 9830:M 19 Dec 07:41:43.520 * Ready to accept connections


```
OK. Sau khi Redis đã lên. Ta bắt đầu vào code. Ta sẽ tạo 1 file: subscribe.py. Tạo kết nối với redis bằng cách sử dụng library redis-py

```
import redis


HOST = 'localhost'
PORT = '6379'

if __name__ == '__main__':
    r = redis.Redis(host=HOST, port=PORT)

```

Trong readme của redis-py, ta có thể dễ dàng tìm thấy hướng dẫn tạo Pubsub: https://github.com/andymccurdy/redis-py

```
>>> r = redis.Redis(...)
>>> p = r.pubsub()
...

```

Ok. Ở demo, ta sẽ subscribe một channel tên là: test:

```
import redis


HOST = 'localhost'
PORT = '6379'
CHANNEL = 'test'

if __name__ == '__main__':
    r = redis.Redis(host=HOST, port=PORT)
    pub = r.pubsub()
    pub.subscribe(CHANNEL)
```

Sau khi subscribe rồi, ta tiếp tục handle việc nhận data. Đọc trong document ta có thể lấy data như sau:

```
>>> p.get_message()
{'pattern': None, 'type': 'subscribe', 'channel': 'test', 'data': 1}

```

Áp dụng vào demo:

```
import redis
import time

HOST = 'localhost'
PORT = '6379'
CHANNEL = 'test'

if __name__ == '__main__':
    r = redis.Redis(host=HOST, port=PORT)
    pub = r.pubsub()
    pub.subscribe(CHANNEL)

    while True:
        data = pub.get_message()
        if data:
            message = data['data']
            if message and message != 1:
                print("Message: {}".format(message))

        time.sleep(1)

```

Ở đây, ta sở dụng while True để việc get data diễn ra liên tục với time sleep là 1 second. Đến đây ta đã tạo xong subscribe.

### Create Pub

Để tạo publisher, ta cũng tạo ra 1 file publish.py. Tương tự như với subscribe.py, ta cũng cần tạo kết nối với Redis:
```
import redis


HOST = 'localhost'
PORT = '6379'

if __name__ == '__main__':
    r = redis.Redis(host=HOST, port=PORT)

```

Tiếp theo, ta sẽ publish data là một text "Helloworld" trên channel test. Ta sẽ có đoạn code dưới đây

```
import redis
import time

HOST = 'localhost'
PORT = '6379'
CHANNEL = 'test'

if __name__ == '__main__':
    r = redis.Redis(host=HOST, port=PORT)
    pub = r.publish(
        channel=CHANNEL,
        message='HelloWorld!'
    )


```

### Demo

- Run subscribe script:
 ```
$ python subscribe.py
```
- Run publish script và follow console đang chạy subscribe script:
```
$ python publish.py
```
Kết qủa là mỗi lần producer đẩy lên 1 data lên channel thì consumer đều nhận được data đó thông qua channel đã đăng ký.

## Demo bằng Redis-CLI

Redis-cli hỗ trợ các command để Pub/Sub rất đơn giản:

- Subscribe

 ```
$ redis-cli
127.0.0.1:6379> SUBSCRIBE channel [channel ...]

 ```

- Publish

```
$ redis-cli                                                                                    
127.0.0.1:6379> PUBLISH channel message

```

Ta sẽ mở 2 console lên: 1 để subscribe và 1 để publish.

Console subscribe: ta sẽ subscribe channel tên là test-console:

```
127.0.0.1:6379> SUBSCRIBE test-console
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "test-console"
3) (integer) 1


```

Console publish: ta sẽ đẩy một vài message lên channel test-console:

```
127.0.0.1:6379> PUBLISH test-console "hello"
(integer) 0
127.0.0.1:6379> PUBLISH test-console "how are you?"
(integer) 0

```

Quay lại console subscribe ta sẽ nhận được message:

```
127.0.0.1:6379> SUBSCRIBE test-console
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "test-console"
3) (integer) 1
1) "message"
2) "test-console"
3) "hello"
1) "message"
2) "test-console"
3) "how are you?"

```

bạn có thể tham thảo thêm một vài dự san sử dụg Pub/Sub dùng java  [tại đây](https://github.com/search?q=publish+and+subscribe+language%3AJava&type=repositories&l=Java) hoặc bạn có thể tìm thêm các dịch vụ cung cấp  về Cloud Pub/Sub 
