## Tracking System for Analytics

Understanding users is a crucial task for any business. It supports marketing, sales, and improving the product. If most user activities happen on our website or mobile app, We can use Google Analytics - GA. However, we may need our own tracking system for private and custom data. This blog will show you the way. Let's go.

![agenda](https://cdn.hashnode.com/res/hashnode/image/upload/v1657443679778/aA0bfe-1P.png align="left")

# 1. Warm up

This section shows where the tracking system is in our business. Imagine that a camera shop sells new products on its website. There are two main ways for users to know these products: traditional (telesales, broadcast) and digital (Facebook ads, Google ads, etc.). But, how to check the performance of each campaign, e.g. user landing, product purchased.

![Screen Shot 2022-07-10 at 4.17.51 PM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657444697609/NcJScS0CX.png align="left")

On the website, users register to some products, then go to the offline shop to buy registered ones. The tracking system is as below:

![Untitled drawing (1).png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657447675226/iaVqp4mWn.png align="left")

# 2. Existing Solution

If you have no backend devs, this solution is for you. We can embed Google Tag scripts into our websites. It will send events automatically, like landing, clicking, and form submission, to Google Tag Manager. Then, we can redirect events to some platforms for analytics and performance tracking like Facebook (Pixel), Google Ads, and Google Analytics.

![Screen Shot 2022-07-12 at 10.17.58 AM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657596417862/p_WS8qAFf.png align="left")

We can upload offline events as CSV files manually, or we can build a backend system to submit periodically. This solution is fast and good to extract useful insights on web/app activities.

# 3. Our solution

This solution is for private/custom data and can be used along with the above one without any conflicts. Suppose that the current system has frontend and backend (optional). We will add some modification and a new backend service - `event` service.

![Screen Shot 2022-07-12 at 11.58.58 AM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657601964367/7ZkzkaJjy.png align="left")

First, we will map a user with an ID, e.g. UUID or hash of phone, at frontend. For web browser, we put an UUID into local storage if this UUID is not existed (new users). This can be done the same with mobile app. Then, we will embed this UUID in HTTP header, e.g. `X-Request-ID`, of every API requests.

Second, frontend can send some events, e.g. `click` or `submit` to `event` services whenever users interact with the app. Backend will send offsite events, e.g. paid or purchased.

Finally, `event` service will store data into database or stream it to Google Cloud (GCP). In GCP ecosystem, we can query data using SQL and visualize into dashboards with Data Studio. The schema of data requires event name and uuid. The other fields are optional and flexible.

![Screen Shot 2022-07-12 at 3.21.38 PM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657614143023/kkVFKMFWT.png align="left")

# 4. Examples

Because we own data, we can using SQL queries to get any insights we want, including ad-hoc queries. For example, the marketing department asks for the number of users registered to purchase cameras within 1 day after Facebook posts. Moreover, we can build custom dashboards with custom filters too. We can know exactly which section, e.g. home page, recommendation popup, or similar products, that makes users purchase.

![Screen Shot 2022-07-12 at 3.31.45 PM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657614920378/NtYLpDKgq.png align="left")

# 5. Conclusion

Both methods #2 and #3 can be used simultaneously. They will help us understand more about users. With early launching and low cost product, #2 fits our needs perfectly. It also helps us run campaigns in Facebook or Google Ads. If we need deeper insights about product, #3 will be the choice.

![Screen Shot 2022-07-12 at 12.39.02 PM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657604459195/9lRLBSJ_r.png align="left")

Thank you for reading my blog!
