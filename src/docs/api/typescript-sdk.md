---
icon: typescript
---
# TypeScript SDK

::: warning
The TypeScript SDK is under development, APIs might be changed in later versions. Pull requests are welcomed: https://github.com/gorse-io/gorse-js
:::

## Install

::: code-tabs#install

@tab:active npm

```bash
npm install gorsejs
```

@tab yarn

```bash
yarn add gorsejs
```

:::

## Usage

Create a client by the entrypoint and api key.

```js
import { Gorse } from "gorsejs";

const client = new Gorse({ endpoint: "http://127.0.0.1:8087", secret: "api_key" });

await client.insertFeedbacks([
    { FeedbackType: 'star', UserId: 'bob', ItemId: 'vuejs:vue', Timestamp: '2022-02-24' },
    { FeedbackType: 'star', UserId: 'bob', ItemId: 'd3:d3', Timestamp: '2022-02-25' },
    { FeedbackType: 'star', UserId: 'bob', ItemId: 'dogfalo:materialize', Timestamp: '2022-02-26' },
    { FeedbackType: 'star', UserId: 'bob', ItemId: 'mozilla:pdf.js', Timestamp: '2022-02-27' },
    { FeedbackType: 'star', UserId: 'bob', ItemId: 'moment:moment', Timestamp: '2022-02-28' }
]);

await client.getRecommend({ userId: 'bob', cursorOptions: { n: 10 } });
```
