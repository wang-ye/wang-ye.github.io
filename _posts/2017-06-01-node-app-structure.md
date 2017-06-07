---
layout: post
title:  "Reading A Rental Visualizer Node Project"
date:   2017-06-01 17:49:03 -0800
---

Node has gained much popularity recently and I want to understand how to write Node app. Instead of starting with the basic hello-world app, I believe reading some interesting open source project is the quickest way to learn. So here we go: the App to discuss is called [Rental](https://github.com/answershuto/Rental). It is used to visualize available rentals in any China city. The commit hash is [4e9a59a](https://github.com/answershuto/Rental/commit/4e9a59ab7849127022094ac8890d4748b0d0a383).

## Code Structure
The fact `package.json` contains all necessary information from installing dependencies to executing command blows my mind every time.
Running Rental is super easy now:

```bash
npm install && npm start
```

*Rental* is a web application, and it uses [express](https://expressjs.com/) as the web framework. We will start with `express.js`, which details different facets of the app.

```javascript
var controller = require('../app/controllers/rental.server.controller');

module.exports = function(){
    var app = express();

    app.set('view engine','ejs');
    app.use(express.static(__dirname+'/../views'));

    app.use(bodyParser.json());

    require('../app/routes/rental.server.routes')(app);
    controller.init();

    // Express 404 page and error handling are skipped.
    return app;
}
```

In this project, each JS file contains three pieces:

1. Importing other packages with require
2. The actual logic
3. Exporting the logic externally with module.exports

The application logic sets up the view engine and body parser. Also, we can locate the router and controller location. The models, on the other hand, do not exist. Two routers are defined:

```javascript
// In lib/router/route.js
app.route('/')
    .get(function(req,res,next){
        res.sendFile('index.html');
    });

app.route('/rental/getInfos')
    .all(RentalController.getRentalInfos);
```

Let's first dive into the controllers to see the how RentalController.getRentalInfos is coded.

```javascript
// app/controllers/rental.server.controller.js
getRentalInfos(){
    let params = {};

    for(let [k,v] of rentalInfosMap){
        params[k] = v;
    }
    return params;
}
```

Rental details are all stored rentalInfosMap, each entry of which is populated by `analysis` method:

```javascript
// rental.server.controller.js
function analysis(url){
    let html = "";
    http.get(url, function(res){
        res.on('data', function(chuck){
            html += chuck;
        })

        res.on('end', function(){
            let $ = cheerio.load(html);
        $('a.c_333') && $('a.c_333')['0'] 
            && rentalInfosMap.set(url, {
                tel: $('span.tel-num.tel-font').text(),
                price: $('.house-price').text(),
                location: $('a.c_333')[0].children[0].data,
                img: $('#smainPic')['0'].attribs.src,
            })
            // ..
        })
    }
}
```

`Analysis` method issues an HTTP request with a given URL, and extracts certain fields with css selectors.

Finally, the Viewer part! To visualize the houses, an Ajax call is initiated:

```javascript
// index.html
$.ajax({
    'type': 'post',
    'url': '/rental/getInfos',
    'contentType': 'application/json;charset=utf-8',
    'data': JSON.stringify({params: null}),
    success: getInfosSuc ,
    async: true,
    error: getInfosErr ,
})
```
If success, getInfoSuc (omitted for simplicity) is called to draw the maps and houses on Baidu Map, a famous map product in China.

## Summary
This is an experiment to learn new languages (Node here) through reading open source projects. I find it especially useful to quickly understand the code organization and get familiar with the tools. This learning approach also provides directions regarding important packages to start with.