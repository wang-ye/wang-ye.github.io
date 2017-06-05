A Rental Visualizer Node App

Node has gained popularity recently and I want to understand how to write a working and useful Node app. I previously read some Node basics and syntax, but still have no idea how to write a usable App. This time, instead of starting with the basic hello-world app, I prefer to read open source project.
The App we are going to cover is called [Rental](https://github.com/answershuto/Rental) with commit hash [4e9a59a](https://github.com/answershuto/Rental/commit/4e9a59ab7849127022094ac8890d4748b0d0a383).

## An Intro for Rental
This is an App to visualize available rentals in any China city.

## Code Structure
The fact a simple `package.json` contains all information from installing dependencies and execution command really blew my mind every time.
Running Rental is super easy

```bash
npm install && npm start
```

The App is essentially a web application, The web framework is [express](https://expressjs.com/). As a side note, each of the JS file has three pieces:

Importing other packages with require
The actual logic
Exporting the logic externally with module.exports

Looking at `express.js`, it contains three pieces

```javascript
var controller = require('../app/controllers/rental.server.controller');

function(){
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

After setting up the view engine and body parser, we found the the router and controller. Two routers are defined:

package.json to detail both dev and production dependencies and how to run the apps.

```javascript
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

Rental details are retrieved from rentalInfosMap, which is populated by getting a list of URLs, and then extract certain fields.

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

Finally, the froentend part.
To visualize the houses, an Ajax call is initialted.
```javascript

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
If success, getInfoSuc is called to draw the maps and houses on Baidu Map, a famous map product in China.

This is what happens in this Node app.