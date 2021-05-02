---
title: "#4noob:Sơ lược Content Security Policy Part-1"
date: 2020-04-10T17:01:47+07:00
tags: csp, tutorial, noobs
---

### 1. CSP

XSS (Cross Site Scripting ) là một vấn đề bảo mật web rất phổ biến, nó nằm trong top 10 OSWAP

Thực chất nguyên do xảy ra XSS là nguyên lý rất dễ hiểu - một đoạn code độc hại được nhúng vào trang web, theo nhiều cách khác nhau từ việc input dạng lưu trữ (stored xss) hay với một được link tới web có kèm code độc hại (reflected XSS). Biện pháp phòng chống thường được dùng là làm sạch (sanitize) user input, cách này khá hiệu quả tuy nhiên khi trang web có nhiều user input, có thể một số sẽ bị miss thế nên cần có một số biện pháp khác đi kèm, trong đó là ***Content-Security-Policy (CSP)*** . 

CSP thực chất giống như một ***whilelist*** cho phép khai báo các nguồn tin tưởng - web application tin tưởng  để load thông tin từ những nguồn đó, vd: cho phép load scripts của bên thứ ba (như google, facebook ..) trên ứng dụng web của mình và hạn chế scripts từ các nguồn còn lại. 

### 2. Implement CSP 

Có 2 cách triển khai CSP: 

1. Thông qua HTTP response headers
2. Thông qua meta tags trong HTML file

Thông thường triển khai CSP qua HTTP headers khá phổ biến vì dễ triển khai và dễ chỉnh sửa CSP rule. Có thể thực hiện thông qua web server, middleware ...  nếu cần thay đổi CSP thì chỉ cần khai thêm ở phần chương trình handle trang web đó là ok. 

Để hiểu rõ hơn không gì tốt hơn là vào ví dụ thực tiễn 😁

#### Bắt đầu với 1 trang web mini nào:

```
var express = require('express');
var app = express();

app.get('/', function(req, res){
    res.send("Hello " + req.query.name);
    console.log(req.query.name);

})
app.use('/lib',express.static('lib'))
app.use(function(req, res){
    res.status("404");
    res.send("Ops! Not Found");
})


app.listen(3000);
```

Rất dễ để attack XSS :

![image-20200410160031444](/image/csp/1.png)

Giờ để phòng chống XSS như trên ta triển khai CSP, ở đây mình dùng library **simple-csp**

```
var express = require('express');
var csp = require('simple-csp');
var app = express();
var csp_headers = {
    "default-src": ["'self'"]  
};

app.use("/", function(req, res, done){
    csp.header(csp_headers, res);
    done();
})

app.get('/', function(req, res){
    res.send("Hello " + req.query.name);
    console.log(req.query.name);

})
app.use('/lib',express.static('lib'))
app.use(function(req, res){
    res.status("404");
    res.send("Ops! Not Found");
})


app.listen(3000);
```

Ở đây mình dùng [`default-src`](https://content-security-policy.com/default-src/) directive với `self` nghĩa là chỉ cho phép thực hiện script từ các nguồn same origin, same domain và scheme. Tuy nhiên nhìn có vẻ rule này không có tác dụng cho lỗi XSS trên vì script được nhúng trên chính web đó? thực chất CSP coi tất cả *inline script* đều nguy hiên nên sẽ hạn chế theo mặc đinh, trừ khi ta khai báo `unsafe-inline`. 

Bạn có thể xem thên các directive đầy đủ của CSP tại : https://content-security-policy.com/ 

Giờ thì hãy thử attack lại nhé:

![image-20200410161226340](/image/csp/2.png)

It worked !! Thế là ta đã sử dụng CSP để chống XSS thành công 😎. Tuy nhiên, CSP cũng có thể bị bypass nếu rule không chặt chẽ hoặc các nguồn tin tưởng bị lợi dụng để tạo payload attack. 

Giả sử có 1 trang khác trên web page  có thể dùng để inject xss, ta có thể lợi dụng trang đó để tấn công xss trang chính. 

![image-20200410161817739](/image/csp/3.png)

![image-20200410161936900](/image/csp/4.png)

Vì xss payload được load từ trang cùng domain (phù hợp với `self`) và không phải từ chính trang đó (không bị hạn chế bởi `inline` mặc định) => XSS thành công. 

Cuối cùng cũng viết xong bài đơn giản về csp.  😢

The end. 

