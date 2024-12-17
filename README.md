# Appido
Appido Node.js Answer


# Answer 1:
در مدیریت عملیات همزمان asynchronous در Node.js، استفاده از Promises و async/await به‌خوبی می‌تواند مشکلات ناشی از کدنویسی callback-based را حل کند. تجربه‌ای که می‌خواهم توضیح دهم، مربوط به به پروژه‌ای می‌شود که در آن باید دیتای زیادی را از یک API خارجی fetch میکردیم و در MongoDB ذخیره می‌کردیم و بر اساس اون دیتاها یه سری پردازش ها انجام میشد و همزمان برای وب ساکت ارسال میشد.

### Project scenario:
ما در این پروژه نیاز داشتیم که اطلاعات محصولات مختلف را از چندین endpoint مختلف fetch کنیم و آن‌ها را به‌طور همزمان به DB بنویسیم. عملیات شامل مراحل زیر بود:

 - خواندن اطلاعات از API برای هر دسته‌بندی. <br>
 - ذخیره‌سازی دیتاها در پایگاه داده. <br>
 - انجام این عملیات برای تعداد زیادی از درخواست‌ها به‌صورت موازی (تقریباً ۱۰۰+ درخواست). <br>
(بر اساس سوالی که مطرح کردین تا این قسمت فکر کنم مد نظرتون بود ولی چون من داشتم روی اون دیتاها تصمیماتی خاصی میگرفتم و برای وب ساکت میفرستادم، مراحل بیشتری داره) <br>

البته این مورد را یاداور بشم که چون حجم دیتای ارسال از اون سمت برامون خیلی زیاد بود مجبور بودم از thread ها استفاده کنم و با PM2 اینکارو هندل میکردم که این قسمت خودش جزییات زیادی داره. <br>

از مبحث اصلی دور نشیم.<br>

### Promise.all & async/await:

برای انجام همزمان چند عملیات و اجرای موازی درخواست‌ها، از Promise.all استفاده کردیم. دلیل استفاده از این متد این بود که درخواست‌ها مستقل از هم بودند و اجرای همزمان آن‌ها به performance پروژه کمک می‌کرد.


```typescript

const axios = require('axios');

// Fetch function for get data
async function fetchProductData(url: string): Promise<any> {
  try {
    const response = await axios.get(url);
    return response.data;
  } catch (error) {
    console.error(`Error fetching data from ${url}`, error.message);
    throw error;
  }
}

// Async function
async function fetchAllDataAndStore(urls: string[]) {
  try {
    const fetchPromises = urls.map((url) => fetchProductData(url)); // create an array of promise
    const results = await Promise.all(fetchPromises); // run all request at same time
    
    // Save data on DB
    for (const data of results) {
      await storeDataInDatabase(data);
    }

    console.log('All data fetched and stored successfully.');
  } catch (error) {
    console.error('An error occurred during the fetch or store process:', error.message);
  }
}

// For example this function save on DB (we can use Prisma or TypeORM)
async function storeDataInDatabase(data: any): Promise<void> {
  console.log('Storing data:', data);
}

```

### Race Condition Management:
در این پروژه، یکی از مشکلاتی که ممکن بود رخ دهد Race Condition هنگام ذخیره‌سازی داده‌ها در دیتابیس بود. چون درخواست‌ها همزمان اجرا می‌شدند، ممکن بود داده‌های مشابه توسط چند درخواست ذخیره شوند.
<br>


- استفاده از فیلدهای یکتا (Unique Index)

ایجاد یک Unique Index در سطح پایگاه داده، به ما کمک می‌کند که جلوی ورود داده‌های تکراری و همزمان را بگیریم. حتی اگر چند عملیات همزمان تلاش کنند یک داده مشابه را بنویسند، MongoDB به‌طور خودکار یکی را قبول می‌کند و برای دیگری خطا برمی‌گرداند.
```js
db.collection.createIndex({ email: 1 }, { unique: true });
```
 - استفاده از عملیات اتمیک (Atomic Operations)

  به کمک عملیات‌های اتمیک مثل `updateOne` و `findOneAndUpdate` امکان به‌روزرسانی امن داده‌ها را فراهم می‌کند. این عملیات‌ها می‌تواند به کمک اپراتورهایی مثل `$set, $inc, $push` و ... به‌صورت امن و بدون ایجاد Race Condition داده‌ها را تغییر دهند.
  
```typescript
// example for findOneAndUpdate
const result = await Product.findOneAndUpdate(
  { productId: '123' },
  { $setOnInsert: { name: 'Sample Product', price: 100 } },
  { upsert: true, new: true }
);
console.log('Result:', result);
```

- استفاده از Locks به کمک MongoDB Transactions
  با این روش ما میتوانیم تراکنش های به هم مرتبط را در یک بلوک قرار دهیم و اگر خطا داشتیم همه ی عملیات ها رول بک شوند.
```typescript
const session = await mongoose.startSession();

try {
  session.startTransaction();

  const product = await Product.findOne({ productId: '123' }).session(session);
  if (!product) {
    await Product.create([{ productId: '123', name: 'Sample Product', price: 100 }], { session });
  } else {
    await Product.updateOne({ productId: '123' }, { $inc: { price: 10 } }).session(session);
  }

  await session.commitTransaction();
  console.log('Transaction Successful');
} catch (error) {
  await session.abortTransaction();
  console.error('Transaction Failed:', error.message);
} finally {
  session.endSession();
}
```

- استفاده از سیستم صف (Queue)
در مواردی که تعداد زیادی عملیات همزمان باید انجام شود، می‌توان از یک سیستم Queue مثل Bull یا RabbitMQ استفاده کرد(که ما از RabbitMQاستفاده کردیم). سیستم صف درخواست‌ها را صف‌بندی کرده و آن‌ها را یکی‌یکی پردازش می‌کند.

### Deadlock Management:
- از تراکنش‌های کوتاه‌تر و ساده‌تر استفاده کردیم.
- به عملیات پایگاه داده اولویت و ترتیب مشخص دادیم.
- از retry mechanism برای تراکنش‌هایی که با شکست مواجه می‌شدند استفاده کردیم.

### Why Promise.all?
- درخواست‌های مستقل از هم بودند و ترتیب اجرای آن‌ها اهمیتی نداشت.
- نیاز به کارایی بالا داشتیم و نمی‌خواستیم درخواست‌ها به‌صورت سریال اجرا شوند.
- استفاده از Promise.all به ما این امکان را داد که تمام درخواست‌ها به‌طور همزمان اجرا شوند و نتیجه همه‌ی آن‌ها را در یک مرحله دریافت کنیم.

<br><br><br><br>




# Answer 2:

معمولا این سبک پروژه ها حافظه ی زیادی مصرف میکنن و باید Memory Leakها را شناسایی کرد و براش راه حلی در نظر گرفت

### Detect Memory Leak

ما معمولا از Chrome DevTools استفاده میکردیم و بر اساس دیتایی که دریافت میکردیم، استراتژی هایی در نظر میگرفتیم.
برای مثال معمولا دنبال objectهایی بودیم که در طول زمان از بین نمیرفتن و در حافظه باقی میموندن و بیشتر تمرکزمون روی Global Variableها ، EventListenerها، Closureها و Timerها این موارد بود.

### How Increase Performance? 
برای اینکه سیستم را بهینه کنیم، این موارد را توی کدها رعایت میکردی
- پاکسازی Event Listeners:
```typescript
emitter.removeListener('myEvent', myHandler);
```
- مدیریت Timers:
```typescript
clearInterval(timer);
```
- مدیریت Closures:
```typescript
const closure = createClosure();
closure = null;
```

<br><br><br><br>

# Answer 3

در این پروژه‌ که نیاز به پردازش‌ سنگین CPU داشتیم، از Worker Threads استفاده کردیم. استفاده از Worker Threads زمانی ضروری شد که Event Loop به دلیل عملیات‌های سنگین، دچار بلاک شدن می‌شد و باعث می‌شد که عملکرد اپلیکیشن و response به بقیه requestها کندتر شود.
### Why use Worker Threads

چون نودجی اس  Single Threaded هست، تمام پردازش‌ها از طریق Event Loop انجام میشه که این ساختار برای پردازش‌های I/O مانند خواندن و نوشتن فایل، ارتباط با دیتابیس خیلی کارآمد است. اما زمانی که پردازش‌های CPU-Intensive مانند رمزنگاری، تحلیل دیتای سنگین یا پردازش فایل‌های بزرگ نیاز باشد، Event Loop مسدود می شود.

به همین دلیل برای جلوگیری از:

- بلاک شدن Event Loop
- کاهش کارایی اپلیکیشن
- افزایش پاسخ‌گویی به درخواست‌ها
از Worker Threads استفاده کردیم. هر Worker Thread یک Thread جداگانه برای اجرای محاسبات CPU فراهم می‌کند و به ما اجازه می‌دهد از تمام هسته‌های CPU استفاده کنیم.

<br> <br>
ارتباط بین Main Thread و Worker Threads به کمک پیام‌ها (Message Passing) انجام میدیم.
در این روش از متد postMessage برای ارسال ‌دیتاها و on('message') برای دریافت پیام استفاده می‌کنیم.

```typescript
// main.js
const { Worker } = require('worker_threads');

function runWorker(filePath, input) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(filePath, { workerData: input });

    worker.on('message', (result) => {
      resolve(result);
    });

    worker.on('error', reject);

    worker.on('exit', (code) => {
      if (code !== 0) {
        reject(new Error(`Worker stopped with exit code ${code}`));
      }
    });
  });
}

(async () => {
  try {
    console.log('Main Thread: شروع پردازش');
    const result = await runWorker('./worker.js', { num: 100000000 });
    console.log('Result from Worker Thread:', result);
  } catch (error) {
    console.error('Error:', error);
  }
})();

```

```typescript
// worker
const { parentPort, workerData } = require('worker_threads');

function heavyComputation(limit) {
  let sum = 0;
  for (let i = 0; i < limit; i++) {
    sum += i;
  }
  return sum;
}

const result = heavyComputation(workerData.num);

// ارسال نتیجه به Main Thread
parentPort.postMessage(result);

```

<br><br><br><br>

# Answer 4

این مورد را کار نکردم

# Answer 5
این مورد هم به اینصورت که CQRS کار کرده باشم نبود، یه سری اطلاعات دارم ولی این پروژه ایی که کار کردم در حدی نبود که بخوام از این روش استفاده کنم.

# Answer 6

این سوال متوجه منظورتون نشدم و شاید هم کار نکرده باشم. یه سری کارها هست که برای دیپلوی پروژه انجام میدیم که این موارد هم تسلط زیاد ندارم و بر اساس نیاز و آزمون و خطا انجام میدیم. حالا اگر جزییات بیشتری بود من درخدمتم
