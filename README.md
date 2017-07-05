# Test
## Aliyun test
module.exports = function (grunt) {

    require('load-grunt-tasks')(grunt);
    if (typeof String.prototype.startsWith != 'function') {
     String.prototype.startsWith = function (prefix){
      return this.slice(0, prefix.length) === prefix;
     };
    }


    var request = require('request');
    var https = require('https');
    var proxyRequest = function(req, res, next){
        var url = req.url;
        grunt.log.writeln(url);

        var headers = req.headers;
        headers['Content-Type'] = 'application/json';
        headers.host = 'localhost';

        var options = {
            url: 'https://127.0.0.1:9002/provider/query/json/language/list',
            method: req.method,
            headers: headers,
            // cert: grunt.file.read('../server/server.crt').toString(),
            // key: grunt.file.read('../server/server.key').toString(),
            ca: grunt.file.read('../server/server.pem').toString(),
            checkServerIdentity: function (host, cert) {
                return undefined;
              }

        };

        grunt.log.writeln('#############1');
        grunt.log.writeln(Object.keys(res));
        request(options, function (error, response, body) {
            grunt.log.writeln(error);
            grunt.log.writeln(response.toJSON());
            // grunt.log.writeln(Object.keys(response));
            grunt.log.writeln(body);
            res.end(body);
        });
    }
    var serveStatic = require('serve-static'),
        /**
         * 本地模式: API_NAME配置成 api，DEVELOP_MODE配置为 true
         * 代理模式: API_NAME配置成provider，DEVELOP_MODE配置为 false
         * */
        API_NAME = 'provider',
        proxyRewrite = {
            '^/provider/': '/provider/'
        },

        getReplaceOptions = function () {
            var DEVELOP_MODE = false,
                CONTEXT_PATH = '';
            return {
                patterns: [
                    {
                        match: 'DEVELOP_MODE',
                        replacement: DEVELOP_MODE
                    },
                    {
                        match: 'CONTEXT_PATH',
                        replacement: CONTEXT_PATH
                    },
                    {
                        match: 'API_NAME',
                        replacement: API_NAME
                    }
                ]
            }
        };

    grunt.initConfig({
            pkg: grunt.file.readJSON('package.json'),
            copy: {
                js: {
                    expand: true,
                    cwd: 'src/',
                    src: ['**/*'],
                    dest: 'dist/'
                }
            },
            watch: {
                options: {
                    livereload: true
                },
                js: {
                    files: 'src/**/*.js',
                    tasks: ['copy:js','replace:js']
                }
            },
            connect: {
                options: {
                    port: '9001',
                    hostname: '127.0.0.1',
                    protocol: 'http',
                    open: false,
                    base: {
                        path: './',
                        options: {
                            index: 'html/index.html'
                        }
                    },
                    livereload: true
                },
                proxies: [
                    {
                        context: '/' + API_NAME,
                        host: '127.0.0.1',
                        port: '9002',
                        https: true,
                        changeOrigin: true,
                        rewrite: proxyRewrite
                    }
                ],
                default: {},
                proxy: {
                    options: {
                        middleware: function (connect, options) {
                            // if (!Array.isArray(options.base)) {
                            //     options.base = [options.base];
                            // }

                            // // Setup the proxy
                            // var middlewares = [require('grunt-connect-proxy/lib/utils').proxyRequest];

                            // // Serve static files.
                            // options.base.forEach(function (base) {
                            //     middlewares.push(serveStatic(base.path, base.options));
                            // });

                            // // Make directory browse-able.
                            // /*var directory = options.directory || options.base[options.base.length - 1];
                            //  middlewares.push(connect.directory(directory));
                            //  */
                            // return middlewares;

                            return [proxyRequest];
                        }
                    }
                }
            },
            replace: {
                options: getReplaceOptions(),
                js: {
                    expand: true,
                    cwd: 'dist/js/',
                    src: ['**/*.js'],
                    dest: 'dist/js/'
                }
            }
        }
    );

    grunt.registerTask('staticServer', '启动静态服务......', function () {
        grunt.task.run([
            'copy',
            'replace',
            'connect:default',
            'watch'
        ]);
    });

    grunt.registerTask('proxyServer', '启动代理服务......', function () {
        grunt.task.run([
            'copy',
            'replace',
            'configureProxies:proxy',
            'connect:proxy',
            'watch'
        ]);
    });
};
