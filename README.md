# Node.js Async Function Best Practices (Những kinh nghiệm hay với hàm _async_ trong NodeJS)
Kể từ phiên bản **NodeJS** 7.6, **NodeJS** "chuyên chở" với một phiên bản _engine V8_ mới (V8 version 5.5) được hỗ trợ xử lý các hàm bất đồng bộ với từ khóa _async_. Thêm vào đó, Ngày 31/10/2017 vừa qua NodeJS 8 chính thức trờ thành **LTS version** (Long Term Support - phiên bản được hỗ trợ dài hạn) của LTS team NodeJS. Vì vậy, không có lý do gì không bắt đầu sử dụng các hàm _async_ trong cấu trúc code của bạn. Trong bài viết này, giới thiệu ngắn gọn cho bạn hàm _async_ là gì, và cái cách mà nó sẽ làm thay đổi cách chúng ta viết các ứng dụng **NodeJS**.

## Hàm ASYNC là gì?

Các hàm _async_ cho phép bạn viết các đoạn code dựa trên **Promise** trông như là nó được xử lý đồng bộ vậy (lần lượt từ trên xuống dưới). Một lần bạn định nghĩa một hàm sử dụng từ khóa _async_ (gọi là _async function_), sau đó bạn sử dụng từ khóa _await_ bên trong thân hàm đó. 

Khi _async function_ được gọi, nó trả về một **Promise**. Khi _async function_ trả về một giá trị, **Promise** nhận trạng thái _fullfilled_, còn nếu _async function_ trả về _error_, **Promise** nhận trạng thái _rejected_.

Từ khóa _await_ có thể được sử dụng để đợi một **Promise** (cái mà được resolved và trả về trạng thái _fullfilled_). Nếu một giá trị (_value_) truyền tới từ khóa _await_ không phải là một **Promise**, nó chuyển đổi _value_ thành một resolve Promise.

```js
const rp = require('request-promise')
async function main () {
  const result = await rp('https://google.com')
  const twenty = await 20
  
  // sleeeeeeeeping  for a second 💤
  await new Promise (resolve => {
    setTimeout(resolve, 1000)
  })
  return result
}
main()
  .then(console.log)
  .catch(console.error)

```
## Di trú (Migrating) tới các hàm _async_

Nếu các ứng dụng **NodeJS** của bạn đã đang sử dụng **Promises**, bạn chỉ việc bắt đầu _awaiting_ các **Promise** của bạn thay vì nối chuỗi (chaining) chúng. 

Nếu các ứng dụng của bạn đang xây dựng sử dụng _callbacks_, việc chuyển sang các hàm 
_async_ nên tiến hành dần dần. Bạn có thể bắt đầu thêm các tình năng mới bằng việc sử dụng kỹ thuật mới này (_async/await_). Nếu bạn phải sử dụng các phần cũ hơn của ứng dụng, bạn có thể đơn giản bọc (wrap) chúng trong các **Promises**.
Để làm vậy, bạn có thể sử dụng phương thức xây dựng sẵn (built-in method) _util.promisify_

```js
    const util = require('util')
const {readFile} = require('fs')
const readFileAsync = util.promisify(readFile)
async function main () {
  const result = await readFileAsync('.gitignore')
  return result
}
main()
  .then(console.log)
  .catch(console.error)
```
## Thực hành tốt hơn cho các hàm _**async**_

### #Sử dụng các hàm _async_ với _express_

Bởi vì _express_ hỗ trợ **Promises**, sử dụng các hàm _async_ với _express_ đơn giản như sau: 
```js
const express = require('express')
const app = express()
app.get('/', async (request, response) => {
  // awaiting Promises here
  // if you just await a single promise, you could simply return with it,
  // no need to await for it
  const result = await getContent()
  response.send(result)
})
app.listen(process.env.PORT)
```

Chỉnh sửa 1: Như là Keith Smith đã chỉ ra, ví dụ trên có 1 vấn đề phát sinh nghiêm trọng - Nếu như **Promise** nhận trạng thái _rejected_, _the express route handle_ (xử lý điều hướng của _express_) sẽ bị treo, bởi vì không có xử lý lỗi được gọi ra.

Để xử lý vấn đề đó, bạn nên bọc (wrap) các xử lý bất đồng bộ đó trong một hàm xử lý lỗi:
```js
  const awaitHandlerFactory = (middleware) => {
  return async (req, res, next) => {
    try {
      await middleware(req, res, next)
    } catch (err) {
      next(err)
    }
  }
}
// and use it this way:
app.get('/', awaitHandlerFactory(async (request, response) => {
  const result = await getContent()
  response.send(result)
}))
```
## Parallel execution (Thực thi song hành)

Tưởng tượng bạn đang làm việc trên một cái gì đó tương tự như, khi một tác vụ cần 2 đầu vào (_input_), một từ database, và một từ dịch vụ bên ngoài khác:

```js
  async function main () {
  const user = await Users.fetch(userId)
  const product = await Products.fetch(productId)
  await makePurchase(user, product)
}
```

Trong trường hợp này, mọi việc diễn ra như sau:

  * Đầu tiên là lấy về tài nguyên _user resource_ (thông tin về _user_),
  * sau đó lấy về _product resource_ (thông tin về sản phẩm),
  * cuối cùng là thực thi hàm _makePurchase_

Như bạn thấy là, bạn có thể làm 2 việc đầu một cách song hành (parallel), bởi vì chúng không có sự phụ thuộc nào vào nhau cả. Để làm điều này, bạn nên sử dụng phương thức _Promise.all_ :
```js
async function main () {
  const [user, product] = await Promise.all([
    Users.fetch(userId),
    Products.fetch(productId)
  ])
  await makePurchase(user, product)
}
```
Trong một vài trường hợp, bạn chỉ cần kết quả trả về của _resolving Promise_ nhanh nhất - trong trường hợp này, bạn có thể sử dụng phương thức Promise.race.

## Error handling (Xử lý lỗi)

Xem xét code ví dụ sau đây:

```js
async function main () {
  await new Promise((resolve, reject) => {
    reject(new Error('💥'))
  })
}
main()
  .then(console.log)
```

Khi chạy đoạn code trên, bạn sẽ nhận được một thông báo trong _terminal_ của bạn kiểu như:

```
(node:69738) UnhandledPromiseRejectionWarning: Unhandled promise rejection (rejection id: 2): Error: 💥
(node:69738) [DEP0018] DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.
```

Trong các bản mới hơn của **NodeJs**, Nếu như _Promise rejection_ không được xử lý, nó sẽ mang xuống toàn bộ tiến trình **NodeJS**. Bởi vậy, bạn nên sử dụng các khối _try-catch_ khi cần thiết:

```js
const util = require('util')
async function main () {
  try {
    await new Promise((resolve, reject) => {
      reject(new Error('💥'))
    })
  } catch (err) {
    // handle error case
    // maybe throwing is okay depending on your use-case
  }
}
main()
  .then(console.log)
  .catch(console.error)
```

## More complex control flows(Các xử lý phức tạp hơn)

Một trong các thư viện xử lý quá trình bất đồng bộ đầu tiên cho NodeJS là _async_ được tạo bởi Caolan McMahon. Nó cung cấp các hỗ trợ bất đồng bộ, như là: 

  * mapLitmit,
  * filterLimit,
  * concatLimit,
  * or priorityQueue
Nếu bạn không muốn "phát minh lại bánh xe" và thực hiện lại các logic tương tự, và bạn cũng muốn phụ thuộc vào thư viện _battle-tested_ có 50 triệu lượt tải về một tháng, bạn có thể đơn giản sử dụng lại những hàm này với các hàm _async_ cũng tốt, sử dụng phương thức _util.promisify_ :
```js
reinvent the wheel and implement the same logic again, and you also want to depend on a battle-tested library downloaded 50 million times a month, you can simply reuse these functions with async functions as well, using the util.promisify method:

const util = require('util')
const async = require('async')
const numbers = [
  1, 2, 3, 4, 5
]
mapLimitAsync = util.promisify(async.mapLimit)
async function main () {
  return await mapLimitAsync(numbers, 2, (number, done) => {
    setTimeout(function () {
      done(null, number * 2)
    }, 100)
  })
}
main()
  .then(console.log)
  .catch(console.error)
```

## Next up

Tôi hi vọng bạn thích bài viết này và học được nhiều thứ! (:smile). 

Dẫn nguồn: [https://nemethgergely.com/async-function-best-practices/](https://nemethgergely.com/async-function-best-practices/) 
