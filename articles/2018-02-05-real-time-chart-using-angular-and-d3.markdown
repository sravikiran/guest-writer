---
layout: post
title: "Realtime Charts using Angular and D3"
description: "Learn how to build a multi-line chart using D3 and Angular and how to update it in realtime."
date: "2018-02-05 08:30"
author:
  name: "Ravi Kiran"
  url: "sravi_kiran"
  mail: "yuvakiran2009@gmail.com"
  avatar: "https://twitter.com/sravi_kiran/profile_image?size=original"
tags:
- javascript
- angular
- node
- socketio
- d3
related:
- 2017-11-15-an-example-of-all-possible-elements
---

**TL;DR:** Charts create some of the most catchy sections on any business applications and a chart that updates in realtime is a huge value add for the users. Here we will see how to create such charts using Angular and D3. The source code of this article is available in this [GitHub repository](https://github.com/sravikiran/angular-d3-chart/).

## Introduction
With evolution of the web, needs of its users are also increasing. The capabilities of the web in the present era can be used to build very rich interfaces. The interfaces may include widgets in the dashboards, huge tables with incrementally loading data, different types of charts and anything that you can think of. Thanks to the technologies like WebSockets, users want to see the UI updated as early as possible. This is a good problem for us to solve.

This article will build a virtual market application that shows a d3 multi-line chart. That chart consumes data from a Node.js backend consisting of an Express API and SocketIO to get this data in realtime and update the chart whenever it receives a new piece.

## Creating a Virtual Market Server
The demo app we are going to build consists of two parts. One is a Node.js server serving market data and the other is an Angular application consuming the data. As stated, the server will consist of an express API and a socket io endpoint to serve the data continuously.

Create a folder and name it `virtual-market`. Inside this folder, create a sub folder named `server`. Open a command prompt in the `server` folder and run the following command to generate the `package.json` file:

```bash
npm init -y
```

This command generates the `package.json` file with default configuration. Then run the following command to install the required packages for the server:

```bash
npm install express moment socket.io
```

Once they are installed, we are good to build the server.

### Building an Express API
Add a new file and name it `market.js`. This file will be used like a utility. It will contain the data of a virtual market and it will contain a method to update the data. For now, we will add the data alone and the method will be added while creating the socket.io endpoint. Add the following code to this file:

```js
const moment = require("moment");

let marketPositions = [
  {
    "date": "10-05-2012",
    "close": 68.55,
    "open": 74.55
  },
  // You can add a few more records
];

module.exports = {
  marketPositions
};
```

If you need more data in the `marketPositions` array, copy it from the demo code. I kept just one record in the array to fit it in the article.

Add another file and name it `index.js`. This file will do all the Node.js work required. For now, we will add the code to create an express REST endpoint to serve the data. Add the following code to the file `index.js`.

```js
const app = require('express')();
const http = require('http').Server(app);
const io = require('socket.io')(http);
const market = require('./market');

const port =  3000;

app.use((req, res, next) => {
  res.header('Access-Control-Allow-Origin', '*');
  res.header('Access-Control-Allow-Headers', 'Origin, X-Requested-With, Content-Type, Accept');
  next();
});

app.get('/api/market', (req, res) => {
  res.send(market.marketPositions);
});

http.listen(port, () => {
  console.log(`Listening on *:${port}`);
});
```

After saving this file, you can start the server. Run the following command to start the server:

```bash
node index.js
```

This command starts the Node.js server on the port 3000. Once the server starts, you can visit the URL [http://localhost:3000/api/market](http://localhost:3000/api/market) to see the market updates on last few days.

### Adding Socket IO to Serve Data in Realtime
We need to simulate a realtime market by updating the market data once in every 5 seconds. For this, we will add a method to the file `market.js` and this method will be called from the socket.io endpoint to be created in `index.js`. Open the file `market.js` and add the following code to it:

```js
let counter =  0;

function updateMarket() {
  let diff = Math.floor(Math.random() *  1000) /  100;
  let lastDay = moment(marketPositions[0].date, "DD-MM-YYYY").add(1, "days");
  let open;
  let close;

  if (counter %  2  ===  0) {
    open = marketPositions[0].open + diff;
    close = marketPositions[0].close + diff;
  }
  else {
    open = Math.abs(marketPositions[0].open - diff);
    close = Math.abs(marketPositions[0].close - diff);
  }

  marketPositions.unshift({
    date: lastDay.format("DD-MM-YYYY"),
    open,
    close
  });
  counter++;
}

```

The `updateMarket` method generates a random number and it either adds this value to the last market value or subtracts this value from the last market value to generate some randomness in the figures. Then it adds this entry to the `marketPositions` array. Modify the export statement to include the `updateMarket` method as well.

```js
module.exports = {
  marketPositions,
  updateMarket
};
```

Open the file `index.js`, we will now add a socket.io connection to this file. The socket.io connection will call the `updateMarket` method after every 5 seconds to update the market data and will emit an update on the socket.io endpoint to update the latest data to all the connections. The following snippet shows the code to be added for this:

```js
setInterval(function () {
  market.updateMarket();
  io.sockets.emit('market', market.marketPositions[0]);
}, 5000);

io.on('connection', function (socket) {
  console.log('a user connected');
});
```

Now the server is ready and we can start building the Angular client to use this. To keep the server running, issue the command `node index.js`, if you haven't done it yet.

## Building Angular Application
### Setting up the application
To generate the Angular application, you can use Angular CLI. There are two ways to do it. One is to install the CLI globally and use it and the other is to use npx. I like using npx for this, because it avoids the need to install the package globally and it works with the packages installed in an application as well. Incase if you are not aware of [npx](https://www.npmjs.com/package/npx), it is a command line utility that gets installed with npm version 5.2 and above. npx makes the job of using the global npm packages easier by removing the need of installing the package globally. If you want to use npx, make sure that you have npm 5.2 or above installed. Go to the `virtual-market` folder on a command prompt and run the following command to generate the project:

```bash
npx @angular/cli new angular-d3-chart
```

Once the project is generated, we need to install the d3 and socket.io client libraries. Move to the project folder on the command prompt and run the following command to install these libraries:

```bash
npm install d3 socket.io-client
```

As we will be using these libraries with TypeScript, it is good to have their typings installed. Run the following command to install the typings:

```bash
npm i @types/d3 @types/socket.io-client -D
```

Now the setup process is done and you can run the application to see if everything is fine. Run the following command to start the application:

```bash
npm start
```

Open a browser and change the URL to [http://localhost:4200](http://localhost:4200) to see the default page.

### Building a component to display multi-line chart
Now that the application setup is ready, let's add the required code to it. We will be adding a component to display a multiline d3 chart and the chart will use the data served by the Node.js server. As first thing, let's create a service to fetch the data. For now, the service will consume the REST API to get the stock data. We will consume realtime data from the socket.io endpoint later. Run the following command to add a file for this service:

```bash
npx ng generate service market-status
```

Or you could use the shorter form of this command:

```bash
npx ng g s market-status
```

To consume the REST APIs, we need the `HttpClient` service from the module `HttpClientModule`. The module `HttpClientModule` has to be imported into the application's module for this. Open the file `app.module.ts` and change it as shown below:

```typescript
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { HttpClientModule } from '@angular/common/http';

import { AppComponent } from './app.component';

@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule,
    HttpClientModule
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

The changes made in this file are marked with numbered comments. The `HttpClientModule` is imported and it is added to the `imports` section of the module.


Open the file `market-status.service.ts` on your editor and add the following code to it:

```typescript
import { Injectable } from  '@angular/core';
import { HttpClient } from  '@angular/common/http';
import { Subject, Observable } from  'rxjs';

import  *  as socketio from  'socket.io-client';

import { MarketPrice } from  './market-price';

@Injectable({
  providedIn: 'root'
})
export class MarketStatusService {
  
  private baseUrl =  'http://localhost:3000';
  constructor(private httpClient: HttpClient) { }

  getInitialMarketStatus() {
    return this.httpClient.get<MarketPrice[]>(`${this.baseUrl}/api/market`);
  }
}
```

The variable `socketio` imported in the above file will be used later to fetch data from the Socket IO endpoint.

The `MarketStatusService` uses the class `MarketPrice` for the structure of the data received from the API. Let's create this class now. Add a new file named `market-price.ts` to the `app` folder and add the following code to it:

```typescript
export  class MarketPrice {
  open: number;
  close: number;
  date: string | Date;
}
```

Add a new component to the application, this will show the multi-line d3 chart. The following command adds this component:

```bash
npx ng g c market-chart
```

Open the file `market-chart.component.html` and replace the default content in this file with the following:

```html
<div #chart></div>
```

The d3 chart will be rendered inside this div element. As you see, we created a local variable for the div element, it will be used to get the reference of the element in the component class. This component will not use the `MarketStatusService` to fetch data. Instead, it will accept the data as input. This is done to make the `market-chart` component reusable. For this, the component will have an `Input` field and the value to this field will be passed from the `app-root` component. The component will use the `ngOnChanges` lifecycle hook to render the chart whenever there is change in the data and it will use the `OnPush` change detection strategy to ensure that the chart is re-rendered only when the input changes.

Open the file `market-chart.component.ts` and add the following code to it:

```typescript
import { Component, OnChanges, Input, ElementRef, ChangeDetectionStrategy, ViewChild } from '@angular/core';
import * as d3 from 'd3';

import { MarketPrice } from '../market-price';

@Component({
  selector: 'app-market-chart',
  templateUrl: './market-chart.component.html',
  styleUrls: ['./market-chart.component.css'],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class MarketChartComponent implements OnChanges {
  @ViewChild('chart')
  chartElement: ElementRef;

  parseDate = d3.timeParse('%d-%m-%Y');

  @Input()
  marketStatus: MarketPrice[];

  private svgElement: HTMLElement;
  private chartProps: any;

  constructor() { }

  ngOnChanges() { }

  formatDate() {
    this.marketStatus.forEach(ms => {
      if (typeof ms.date === 'string') {
        ms.date = this.parseDate(ms.date);
      }
    });
  }
}
```

Now the `MarketChartComponent` class has everything required to render the chart. In addition to the local variable for the div and the lifecycle hook, the class has a few fields that will be used while rendering the chart. The `parseDate` method converts string value to date. It is used by the `formatData` method. The private fields `svgElement` and `chartProps` will be used to hold reference of the SVG element and the properties of the chart respectively. These fields would be quite useful to re-render the chart.

Add the following method to the `MarketChartComponent` to build the chart:

```typescript
 buildChart() {
  this.chartProps = {};
  this.formatDate();

  // Set the dimensions of the canvas / graph
  var margin = { top: 30, right: 20, bottom: 30, left: 50 },
    width = 600 - margin.left - margin.right,
    height = 270 - margin.top - margin.bottom;

  // Set the ranges
  this.chartProps.x = d3.scaleTime().range([0, width]);
  this.chartProps.y = d3.scaleLinear().range([height, 0]);

  // Define the axes
  var xAxis = d3.axisBottom(this.chartProps.x);
  var yAxis = d3.axisLeft(this.chartProps.y).ticks(5);

  let _this = this;

  // Define the line
  var valueline = d3.line<MarketPrice>()
    .x(function (d) {
      if (d.date instanceof Date) {
        return _this.chartProps.x(d.date.getTime());
      }
    })
    .y(function (d) { console.log('Close market'); return _this.chartProps.y(d.close); });

  // Define the line
  var valueline2 = d3.line<MarketPrice>()
    .x(function (d) {
      if (d.date instanceof Date) {
        return _this.chartProps.x(d.date.getTime());
      }
    })
    .y(function (d) { console.log('Open market'); return _this.chartProps.y(d.open); });

  var svg = d3.select(this.chartElement.nativeElement)
    .append('svg')
    .attr('width', width + margin.left + margin.right)
    .attr('height', height + margin.top + margin.bottom)
    .append('g')
    .attr('transform', `translate(${margin.left},${margin.top})`);

  // Scale the range of the data
  this.chartProps.x.domain(
    d3.extent(_this.marketStatus, function (d) {
      if (d.date instanceof Date)
        return (d.date as Date).getTime();
    }));
  this.chartProps.y.domain([0, d3.max(this.marketStatus, function (d) {
    return Math.max(d.close, d.open);
  })]);

  // Add the valueline2 path.
  svg.append('path')
    .attr('class', 'line line2')
    .style('stroke', 'green')
    .style('fill', 'none')
    .attr('d', valueline2(_this.marketStatus));

  // Add the valueline path.
  svg.append('path')
    .attr('class', 'line line1')
    .style('stroke', 'black')
    .style('fill', 'none')
    .attr('d', valueline(_this.marketStatus));


  // Add the X Axis
  svg.append('g')
    .attr('class', 'x axis')
    .attr('transform', `translate(0,${height})`)
    .call(xAxis);

  // Add the Y Axis
  svg.append('g')
    .attr('class', 'y axis')
    .call(yAxis);

  // Setting the required objects in chartProps so they could be used to update the chart
  this.chartProps.svg = svg;
  this.chartProps.valueline = valueline;
  this.chartProps.valueline2 = valueline2;
  this.chartProps.xAxis = xAxis;
  this.chartProps.yAxis = yAxis;
}
```

Refer to the comments added before every section in the above method to understand what it does. It has to be called from the `ngOnChanges` lifecycle hook. Change code in this method as follows:

```typescript
ngOnChanges() {
  if (this.marketStatus) {
    this.buildChart();
  }
}
```

Now we need to use this component in the `app-root` component to see the chart. Open the file `app.component.html` and place the following code in it:

```html
<app-market-chart [marketStatus]="marketStatusToPlot"></app-market-chart>
```

And replace content of the file `app.component.ts` with the following code:

```typescript
import { Component } from  '@angular/core';
import { MarketStatusService } from  './market-status.service';
import { Observable } from  'rxjs/Observable';
import { MarketPrice } from  './market-price';

@Component({
selector: 'app-root',
templateUrl: './app.component.html',
styleUrls: ['./app.component.css']
})
export  class AppComponent {
  title =  'app';
  marketStatus: MarketPrice[];
  marketStatusToPlot: MarketPrice[];

  set MarketStatus(status: MarketPrice[]) {
    this.marketStatus = status;
    this.marketStatusToPlot =  this.marketStatus.slice(0, 20);
  }

  constructor(private marketStatusSvc: MarketStatusService) {

  this.marketStatusSvc.getInitialMarketStatus()
    .subscribe(prices => {
      this.MarketStatus = prices;
    });
  }
}
```

Save these changes and run the application using the `ng serve` command. Visit the URL http://localhost:4200, you will see a page with a chart similar to the following image:

[Figure 1 - image of chart]

### Updating the Chart when the Market has an Update
Now that we have the chart rendered on the page, let's receive the market updates from socket.io and update the chart. To receive the updates, we need to add a listener to the socket.io endpoint in the service `market-status.service.ts`. Open this file and add the following method to it:

```typescript
getUpdates() {
  let socket = socketio(this.baseUrl);
  let marketSub =  new Subject<MarketPrice>();
  let marketSubObservable = Observable.from(marketSub);

  socket.on('market', (marketStatus: MarketPrice) => {
    marketSub.next(marketStatus);
  });

  return marketSubObservable;
}
```

The above method does three important things:

 - Creates a manager for the socket.io endpoint at the given URL
 - Creates an RxJS `Subject` and gets the observable from this subject. The observable is returned from this method so that it could be used by the consumer of the service to listen to the updates
 - The call to `on` method on the socket.io manager adds a listener to the `market` event. The callback passed to this method is called whenever the socket.io event publishes something

This method has to be consumed in the `app-root` component. Open the file `app.component.ts` and modify the constructor as shown below:

```typescript
constructor(private marketStatusSvc: MarketStatusService) {
  this.marketStatusSvc.getInitialMarketStatus()
    .subscribe(prices => {
      this.MarketStatus = prices;

      let marketUpdateObservable =  this.marketStatusSvc.getUpdates();  // 1
      marketUpdateObservable.subscribe((latestStatus: MarketPrice) => {  // 2
        this.MarketStatus = [latestStatus].concat(this.marketStatus);  // 3
      });  // 4
    });
}
```

In the above snippet, the statements marked with the numbers are the new lines added to the constructor. Observe the statement labeled with 3. This statement creates a new array instead of updating the field `marketStatus`. This is done to let the consuming `app-market-chart` component know about the change when we have an update.

The last change we need to do to see the chart working is, updating the chart with the new data. Open the file `market-chart.component.ts` and add the following method to it:

```typescript
updateChart() {
  let _this = this;
  this.formatDate();

  // Scale the range of the data again
  this.chartProps.x.domain(d3.extent(this.marketStatus, function (d) {
    if (d.date instanceof Date) {
      return d.date.getTime();
    }
  }));

  this.chartProps.y.domain([0, d3.max(this.marketStatus, function (d) { return Math.max(d.close, d.open); })]);

  // Select the section we want to apply our changes to
  this.chartProps.svg.transition();

  // Make the changes to the line chart
  this.chartProps.svg.select('.line.line1') // update the line
    .attr('d', this.chartProps.valueline(this.marketStatus));

  this.chartProps.svg.select('.line.line2') // update the line
    .attr('d', this.chartProps.valueline2(this.marketStatus));

  this.chartProps.svg.select('.x.axis') // update x axis
    .call(this.chartProps.xAxis);

  this.chartProps.svg.select('.y.axis') // update y axis
    .call(this.chartProps.yAxis);
}
```

The comments added in the snippet explain what we are doing in it. This method has to be called from the `ngOnChanges` method. Change this method as shown below:

```typescript
ngOnChanges() {
  if (this.marketStatus &&  this.chartProps) {
    this.updateChart();
  }
  else if (this.marketStatus) {
    this.buildChart();
  }
}
```

Now if you run the application, you will see an error on the browser console saying `global is not defined`.

[Figure 2 - console error saying global is not defined]

This is because, Angular CLI 6 removed the global object and socket IO uses it. To fix this, add the following statement to the file `polyfills.ts`:

```typescript
(window as any).global = window;
```

With this, all the changes are done. Save the changes and run the application. Now you will see the graph updating once in every 5 seconds.

## Conclusion
As we saw in this tutorial, the web has capabilities to build very rich applications to show realtime updates to the users in a format that gives an immediate impression about the change in data. Let's use these features to build great experiences for our users!
