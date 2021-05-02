---
title: write up DC-2 (VulnHub)
date: 2019-05-30 14:20:01
tags: play, lab
---

Mình viết bài này chỉ để ghi chép những kiến thức trong quá trình thực hiện các bài expolit. Cho các bạn nào chưa biết thì **Vulnhub** là trang chia sẻ các máy ảo có 'lỗ hổng' để luyện tập khai thác. Bạn có thể tải máy ảo cho bài này tại [vulhub](<https://www.vulnhub.com/entry/dc-2,311/>)

### Cần chuẩn bị:
- Kali
- Virtualbox
- Time 😂

Trước hết import ta import máy ảo **DC-2** vừa tải về vào virtualbox hoặc VM (tùy ý bạn - ở đây mình dùng virtualbox). Ở đây mình sử dụng máy ảo kali để thực hiện tấn công nên sẽ cấu hình cho 2 máy ảo này có cùng 1 interface mạng ảo (vboxnet).

Sau khi đã add xong, ta tiến hành scan mạng để tìm địa chỉ của máy cần tấn công - thực tế có thể đoán được địa chỉ này khi add vào virtualbox rồi (nhưng làm vậy cho đúng quy trình 😅). Ta dùng nmap để scan:

```bash
nmap -sS -v -A T4 192.168.56.*/24
```



(192.68.56.* là dải mạng của interface mạng ảo)

Sau khi scan ta thấy được địa chỉ của máy tấn công cũng như service Apache đang chạy port 80.

![image](/image/dc_2/nmap_2.png)


Theo như nhắc nhở của tác giả ta sẽ cấu hình cho kali nhận dc-2 theo địa chỉ của máy cần tấn công. Chỉnh sửa tệp **/etc/hosts**:

![image](/image/dc_2/hosts.png)

Giờ thì dùng trình duyệt để kết nối tới Apache web server của máy cần tấn công, để xem có thông tin hữu ích nào hay không.

Và có vẻ có vài thứ khác "hay ho":

![image](/image/dc_2/cewl.png)



Có vẻ như ta sẽ phải dùng wordlist cho việc expoit tuy nhiên wordlist này tạo từ việc 'cewl' web. Web server này sử dụng Wordpress do đó, ta cứ dùng **wpscan** để quét thử trước.
```bash
$   wpscan --url dc-2 -e
```

Câu lệnh trên sẽ enumarate ra plugin, theme, check vulnerable, user (nếu có thể)

![image](/image/dc_2/wpscan_1.png)

Oh. và ta có được 3 user từ việc scan. Vậy là có thể thấy được từ gợi ý phía trên ta sẽ burte force password với 3 user trên với wordlist tạo từ cewl. Ta có thể tạo file password như sau:
```bash
$   cewl dc-2 > cewl.txt
```

Ok. Giờ chỉ cần burte force bằng wps luôn:
```bash
wpscan --url dc-2 -U user.txt -P cewl.txt`
```
Và ta được kết quả rất tốt ,  ta dò được 2 mật khẩu của 2 tài khoản tom & jerry ( chắc tác giả thích phim hoạt hình tom&jerry quá !!!)

![image](/image/dc_2/wpscan_2.png)


Giờ thử login bằng 2 tài khoản có được, với tài khoản jerry ta có lời gợi ý tiếp theo:

![image](/image/dc_2/login_wp.png)

Không rõ ràng lắm nhưng đại khái là tác giả nói "nên thử hướng khác". Sau nhiều lần thử tấn công theo nhiều các khác nhau nhằm khai thác vào Wordpress không thành công. Mình thử scan lại port kĩ hơn để tìm xem còn service nào khác không. Câu lệnh scan port bạn có thể xem thêm [ở đây](<https://nmap.org/book/port-scanning-tutorial.html>)

```bash
$   nmap -p0- -v -A -T4 192.168.56.103
```

Và ta thấy được port 7704 có service ssh đang chạy

![image](/image/dc_2/nmap_3.png)

Thử login ssh vào server với 2 tài khoản ta có (nhớ điều chỉnh ssh theo port 7704), thì ta login được vào ***tom***

Ta thấy được có **'flag3.txt'**, dùng **less** (sau khi thử nhiều command: nano, vim, cat ) ta có được gợi ý tiếp theo:

![image](/image/dc_2/flag3.png)



Từ gợi ý trên mục tiêu của ta sẽ cố login trở thành "Jerry " vì có thể jerry sẽ có privilege cao hơn. Tuy nhiên khi thử thao tác một số command ta nhận thấy đây là một [restricted shell](<https://www.tldp.org/LDP/abs/html/restricted-sh.html>) cụ thể ta có thể kiểm tra đó là **rbash**.

![image](/image/dc_2/rbash.png)

Do đó để login thành "jerry", ta cần phải bypass restricted shell này. Có thể dễ dàng tìm kiếm trên google để xem tài liệu bypass hoặc có thể tải [ở đây](https://www.exploit-db.com/docs/english/44592-linux-restricted-shell-bypass-guide.pdf).

Để bypass, cần phải xác định xem các command nào có thể sử dụng để thực hiện bypass dựa vào các câu lệnh đó. Kiểm tra các câu lệnh quen thuộc ta thấy **"ls"** có thể dùng và ta phát hiện được các command:

![image](/image/dc_2/rbash_2.png)

Ở đây mình sẽ thực hiện bypass thông qua **vi** . Mình sẽ set biến **shell** từ vi và execute shell

```
#vi
:set shell=/bin/sh
shell
```

Và tada!! ta đã sử dụng được sh.

![image](/image/dc_2/rbash_4.png)

Tiếp theo hãy chuyển **PATH** sang ***/bin*** để có thể sử dụng **su** chuyển qua user jerry. Giờ thì hãy login dưới quyền của jerry với password mà ta có trước đó.

![image](/image/dc_2/rbash_5.png)

Thế là ta đã lấy được ***flag4***.

![image](/image/dc_2/flag4.png)

Còn lại 1 final flag nữa thôi.  Thông thường thì ta phải có quyền root. ta kiểm ta thử có vào được **/root** hay không.

![image](/image/dc_2/check_permission.png)

Vậy là có lẽ thử thách cuối cùng là phải leo thang đặc quyền lên root. Kiểm tra command root mà jerry có quyền chạy. Nếu có ta có thể lợi dụng để khai thác.

![image](/image/dc_2/su_jerry.png)

À. và ta có lệnh **git** có thể chạy dưới quyền root không cần password (**NOPASS**). Dễ dàng hơn rồi, bạn có thể tra cách thức leo thang ở các chương trình phổ biến trên Linux [https://gtfobins.github.io](https://gtfobins.github.io). Ta thực hiện leo thang từ git thôi nào:

```bash
sudo git help status
!/bin/bash
```

Và ta đã có được flag cuối cùng.

![image](/image/dc_2/final_flag.png)

Cuối cùng chúng ta ta kết thúc **dc-2** lab ở đây. Cảm ơn các bạn đã đọc



#### ==== END =====







