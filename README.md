# UI Unit Testing 

### Getting Started
Clone the repository and install Dependencies
```
git clone
cd ui-unit-testing
bower install
npm install -g http-server
```
Running the app
```
http-server
```
Install Web Component Tester
```
npm install -g web-component-tester
bower install --save-dev web-component-tester
```
Create a wct config file `wct.conf.js`
```
module.exports = {
    verbose: false,
    persistent: false,
    plugins: {
        local: {
            browsers: ['chrome']
        },
        sauce: {
            disabled: true
        }
    }
};
```
By default, WCT will look for a folder called test and runs every test in it. You can run wct using the command
```
wct -p
```
### Unit Testing for Behavior
* Create a test file `test/utility-behavior_test.html`
* Import `browser.js`
```
<script src="../bower_components/web-component-tester/browser.js"></script>
```
* Import the behavior you want to test
```
<link rel="import" href="../utility-behavior.html">
```
* Write unit tests in `script` tag
```
<script>
    suite('utilityBehavior', function () {
        
        test('Sort data by key', function () {
            var data = [
                { city: "San Ramon" },
                { city: "NOLA" }
            ]
            var result = [
                { city: "NOLA" },
                { city: "San Ramon" }
            ]
            assert.deepEqual(utilityBehavior.sort(data, "city"), result);
        });
    
    });
</script>
```
### Unit Testing for Component
* Create a test file `test/px-table_test.html`
* Import `browser.js`
```
<script src="../bower_components/web-component-tester/browser.js"></script>
```
* Import the component you want to test
```
<link rel="import" href="../px-table.html">
```
* Create a fixture
```
<test-fixture id="element">
    <template>
        <px-table></px-table>
    </template>
</test-fixture>
```
* Access the element as follows, by declaring it in setup block you get a new instance of the element for each test 
```
<script>
    suite('px-table', function () {
    
        var el;

        setup(function () {
            el = fixture('element');
        });
    
    });
</script>
```
* Sample test case to show how to access an element inside a component
```
test('px-table has table tag', function () {
    assert.isDefined(el.$$('table'));
});
```
* Sample test case to show how to access a component's property
```
test('default tableData should be empty', function () {
    assert.deepEqual(el.tableData, []);
});
```
* Sample test case to show how to access a component's method
```
test('getColumns -- return tableData keys in an array', function () {
    var tableData = [{
        city: "NOLA",
        state: "LA"
    }]
    assert.deepEqual(el.getColumns(tableData), ["city", "state"]);
});
```
* Sample test case to show how to use flush for async operations
```
test('px-table should have tableData keys as column names', function (done) {
    el.tableData = [{
        city: "NOLA"
    }]
    flush(function () {
        assert.equal(el.$$('thead>tr>th').innerText, "city");
        done()
    })
})
```

### Unit Testing for Service
* Create a test file `test/px-service_test.html`
* Import `browser.js`
```
<script src="../bower_components/web-component-tester/browser.js"></script>
```
* Import the service you want to test
```
<link rel="import" href="../px-service.html">
```
* Create a fixture
```
<test-fixture id="element">
    <template>
        <px-service></px-service>
    </template>
</test-fixture>
```
* Access the element as follows, by declaring it in setup block you get a new instance of the element for each test 
```
<script>
    suite('px-service', function () {
    
        var el;

        setup(function () {
            el = fixture('element');
        });
    
    });
</script>
```
* Create a fakeServer using sinon
```
    suite('px-service', function () {
    
        var svc,
            server;

        setup(function () {
            svc = fixture('element');
            server = sinon.fakeServer.create();
            response = {
                count: 10, 
                entries: [{
                    API: 'API',
                    Description: 'Description',
                    Link: 'Link',
                    Category: 'Category'
                }]
            }
            server.respondWith(
                'GET',
                new RegExp('/data.json'),
                [
                    200,
                    { "Content-Type": "application/json" },
                    JSON.stringify(response)
                ]
            )
        });

        teardown(function () {
            server.restore();
        })
    
    });
```
* Sample test case
```
test('set tableData on response', function (done) {
    svc.makeRequest()
    server.respond();
    flush(function () {
        assert.deepEqual(svc.tableData, response.entries);
        done();
    })
});
```
* Sample test case to show how to create a stub method
```
test('px-service response should be a success', function (done) {
    stub('px-service', {
        handleResponse: function(e) {
            assert.equal(e.detail.status, 200);
            done();
        }
    });
    svc.makeRequest();
    server.respond();
});
```

### Integration Testing / Unit testing components which have dependencies
* Create a test file `test/px-app_test.html`
* Import `browser.js`
```
<script src="../bower_components/web-component-tester/browser.js"></script>
```
* Import the component you want to test
```
<link rel="import" href="../px-app.html">
```
* Create a fixture
```
<test-fixture id="element">
    <template>
        <px-app></px-app>
    </template>
</test-fixture>
```
* Access the element as follows, by declaring it in setup block you get a new instance of the element for each test 
```
<script>
    suite('px-app', function () {
    
        var el;

        setup(function () {
            el = fixture('element');
        });
    
    });
</script>
```
* Create a fake-service component
```
<link rel="import" href="../bower_components/polymer/polymer.html">

<dom-module id="fake-service">
    <template>
        <style>
            :host {
                display: block;
            }
        </style>
    </template>
    <script>
        Polymer({
            is: 'fake-service',
            properties: {
                tableData: {
                    type: Array,
                    notify: true,
                    value: []
                }
            }
        });
    </script>
</dom-module>
```
* Replace px-service with fake-service and set tableData
```
    setup(function () {
        replace('px-service').with('fake-service');
        el = fixture('element');
        el.$$('fake-service').tableData = [{
            API: 'One Drive'
        }, {
            API: 'Google Drive'
        }];
    });
```
* Sample test case to show how to simulate an event and retrieve data from fake-service
```
test('Sort table data on selection', function (done) {
    el.$$('select').value = 'API';
    var event = new Event('input', {
        'bubbles': true,
        'cancelable': true
    });
    el.$$('select').dispatchEvent(event);

    flush(function () {
        assert.equal(el.$$('px-table').$$('td').innerText, 'Google Drive');
        done();
    })
});
```
