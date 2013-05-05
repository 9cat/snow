require('shelljs/global')
config.silent = true
var request = require('request')

var exec = function() {
    var result = global.exec.apply(this, arguments)
    if (result.code) throw new Error(result.output)
    return result.output
}

task('clean', function() {
    rm('-Rf', 'public/*')
})

task('app', [
    'public/head.js',
    'public/scripts.js',
    'public/styles.css',
    'public/index.html'
])

task('dist', [
    'public/head.min.js',
    'public/scripts.min.js',
    'public/styles.min.css',
    'public/index.min.html',
    'public/ripple.txt'
])

directory('public')

var head = [
    'public/raven.min.js'
]

var vendor = [
    'public/jquery-2.0.0.min.js',
    'public/sjcl.js',
    'public/alertify.min.js',
    'public/bootstrap.min.js',
    'public/intercom.js'
]

file('public/jquery-2.0.0.min.js', ['vendor/jquery-2.0.0.min.js'], function() {
    cp(this.prereqs[0], this.name)
})

file('public/jquery-2.0.0.min.js', ['vendor/jquery-2.0.0.min.js'], cpTask)
file('public/sjcl.js', ['vendor/sjcl.js'], cpTask)
file('public/intercom.js', ['vendor/intercom.js'], cpTask)
file('public/alertify.min.js', ['vendor/alertify/alertify.min.js'], cpTask)
file('public/bootstrap.min.js', ['vendor/bootstrap.min.js'], cpTask)
file('public/ripple.txt', ['assets/ripple.txt'], cpTask)
file('public/raven.min.js', ['vendor/raven.min.js'], cpTask)

function cpTask() {
    cp(this.prereqs[0], this.name)
}

file('public/scripts.min.js', ['public/scripts.js'], compressJs)
file('public/head.min.js', ['public/head.js'], compressJs)
file('public/styles.min.css', ['public/styles.css'], compressCss)

file('public/index.html', ['public'], function() {
    var ejs = require('ejs')
    ejs.render(cat('assets/index.ejs'), {
        minify: false
    })
    .to(this.name)
})

file('public/index.min.html', ['public'], function() {
    var ejs = require('ejs')
    ejs.render(cat('assets/index.ejs'), {
        minify: true,
        raven: 'https://8216b316efe943d993cd010a6bf22824@app.getsentry.com/6306',
        segment: 'bc0p8b3ul1'
    })
    .to(this.name)
})

file('public/styles.css', ['public'], function() {
    (cat('vendor/bootstrap-combined.min.css') + '\n' +
    cat('vendor/alertify/alertify.bootstrap.css') + '\n' +
    exec('lessc assets/styles.less'))
    .to(this.name)
})

file('public/scripts.js', ['public'].concat(vendor), function() {
    var v = vendor.reduce(function(p, c) {
        return p + ';' + cat(c)
    }, '')
    , bundle = exec('browserify -d -t ./node_modules/browserify-ejs ./entry.js')
    , scripts = v + ';' + bundle
    scripts.to(this.name)
})

file('public/head.js', ['public'].concat(head), function() {
    head.reduce(function(p, c) {
        return p + ';' + cat(c)
    }, '')
    .to(this.name)
})

function compressCss() {
    var inputFn = this.name.replace(/min\.css$/, 'css')
    exec('cleancss ' + inputFn).to(this.name)
}

function compressJs() {
    var inputFn = this.name.replace(/min\.js$/, 'js')
    exec('uglifyjs ' + inputFn + ' --compress warnings=false --mangle').to(this.name)
}

// hosting locally
task('host', [
    'clean', 'app'
], function() {
    var express = require('express')
    , app = express()
    , server = require('http').createServer(app)
    , proxy = require('http-proxy').createServer(function(req, res, proxy) {
        if (req.url.match(/^\/api\//)) {
            // remove /api prefix
            req.url = req.url.substr(4)
            return proxy.proxyRequest(req, res, {
                host: 'localhost',
                port: 5071
            })
        }
        proxy.proxyRequest(req, res, {
            host: 'localhost',
            port: 5072
        })
    })
    proxy.listen(5073)
    app.use(express.static('public'))
    server.listen(5072)
    console.log('hosting at http://localhost:5073')
    return server
})

// publishing
task('pp', ['publish-prod'])
task('publish-prod', [
    'dist'
], function() {
    var async = require('async')
    , files = {
        'head.min.js': null,
        'scripts.min.js': null,
        'styles.min.css': null,
        'index.min.html': 'index.html',
        'ripple.txt': 'ripple.txt'
    }

    async.forEach(Object.keys(files), function(fn, next) {
        var outName = files[fn] || fn
        jake.exec('scp public/' + fn + ' ubuntu@54.228.224.255:/home/ubuntu/snow-web/public/' + outName, next)
    }, complete)
}, { async: true })

// testing
var server

task('test', ['test-host'], function() {
    jake.exec('mocha-phantomjs -R spec http://localhost:5074/test.html', function(res) {
        server.close()
        complete()
    }, {
        printStdout: true
    })
}, { async: true })

file('public/test.html', ['test/support/tests.html'], cpTask)
file('public/test.css', ['node_modules/mocha/mocha.css'], cpTask)

cp('-f', 'test/support/tests.html', 'build/test/index.html')

task('test-host', ['public/test.js', 'public/test.html'], function() {
    var express = require('express')
    , app = express()
    server = require('http').createServer(app)
    app.use(express.static('public'))
    server.listen(5074)
})

file('public/test.js', ['public'].concat(vendor), function() {
    var deps = vendor.slice()
    deps.push('./node_modules/mocha/mocha.js')

    var v = deps.reduce(function(p, c) {
        return p + ';' + cat(c)
    }, '')
    , bundle = exec('browserify -d -t ./node_modules/browserify-ejs ./test/index.js')
    , scripts = v + ';' + bundle
    scripts.to(this.name)
})
