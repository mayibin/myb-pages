## Gulp

-   yarn init 初始化 package.json
-   yarn add gulp --dev
-   创建 gulpfile.js 入口文件

### myb-pages

> 实现这个项目的构建任务
> 读取流、写入流、异步组合、同步组合、监听任务

```
const { src, dest, series, parallel, watch } = require("gulp");
```

> 清空模块

```
const del = require("del");
```

> gulp 管理模块的组件

```
const loadPlugins = require("gulp-load-plugins");
```

> node 的 path 模块

```
const path = require("path");
```

> 开发服务器模块

```
const browserSync = require("browser-sync");
```

> 加载 gulp 模块

```
const plugins = loadPlugins();
```

> 创建开发服务器

```
const bs = browserSync.create();
const cwd = process.cwd();
```

> 抽象化路径

```
let config = {
    build: {
        src: "src",
        dist: "dist",
        temp: "temp",
        public: "public",
        paths: {
            styles: "assets/styles/*.scss",
            scripts: "assets/scripts/*.js",
            pages: "**/*.html",
            images: "assets/images/**",
            fonts: "assets/fonts/**",
        },
    },
};
```

> 渲染模版数据

```
try {
    const pagesConfig = require(`${cwd}/pages.config.js`);
    config = Object.assign({}, config, pagesConfig);
} catch (e) {}
console.log(config)
```

> 创建清空任务
> 安装 del 模块

```
const delText = () => {
    return del([config.build.dist, config.build.temp]);
};
```

> 创建 js 任务流
> 安装 gulp-babel @babel/core @babel/preset-env 模块

```
const scripts = () => {
    return src(path.join(config.build.src, config.build.paths.scripts), {
        base: config.build.src,
    })
        .pipe(plugins.babel({ presets: [require("@babel/preset-env")] }))
        .pipe(dest(config.build.temp))
        .pipe(bs.reload({ stream: true })); >修改 刷新页面
};
```

> 创建 sass 任务
> 安装 gulp-sass 模块

```
const styles = () => {
    return src(path.join(config.build.src, config.build.paths.styles), {
        base: config.build.src,
    })
        .pipe(plugins.sass({ outputStyle: "expanded" })) >设置花括号显示方式
        .pipe(dest(config.build.temp))
        .pipe(bs.reload({ stream: true }));
};
```

> 创建 html 任务
> 安装 gulp-swig 模块

```
const pages = () => {
    return src(path.join(config.build.src, config.build.paths.pages), {
        base: config.build.src,
    })
        .pipe(plugins.swig({ data: config.data, defaults: { cache: false } })) > 防止模板缓存导致页面不能及时更新
        .pipe(dest(config.build.temp))
        .pipe(bs.reload({ stream: true }));
};
```

> 创建 image 任务
> 安装 gulp-imagemin 模块

```
const images = () => {
    return src(path.join(config.build.src, config.build.paths.images), {
        base: config.build.src,
    })
        .pipe(plugins.imagemin())
        .pipe(dest(config.build.dist));
};
```

> 创建 font 任务

```
const fonts = () => {
    return src(path.join(config.build.src, config.build.paths.fonts), {
        base: config.build.src,
    })
        .pipe(plugins.imagemin())
        .pipe(dest(config.build.dist));
};
```

> 创建额外任务

```
const extra = () => {
    return src(path.join(config.build.public, "**"), {
        base: config.build.public,
    }).pipe(dest(config.build.dist));
};
```

> 创建 服务器任务
> 安装 browser-sync 模块

```
const server = () => {
    >监听 js，css，html 文件变化
    watch(path.join(config.build.src, config.build.paths.scripts), scripts);
    watch(path.join(config.build.src, config.build.paths.styles), styles);
    watch(path.join(config.build.src, config.build.paths.pages), pages);
    >监听image font 资源变化
    watch(
        [config.build.paths.images, config.build.paths.fonts],
        { cwd: config.build.src },
        bs.reload
    );
    >监听额外文件变化
    watch("**", { cwd: config.build.src }, bs.reload);
    >初始化服务器配置
    bs.init({
        notify: false,
        server: {
            baseDir: [
                config.build.temp,
                config.build.dist,
                config.build.public,
            ],
            routes: {
                "/node_modules": "node_modules",
            },
        },
    });
};
```

> 创建 useref 任务
> 安装 gulp-useref gulp-if gulp-uglify gulp-clean-css gulp-htmlmin 模块

```
const useref = () => {
    return (
        src(config.build.paths.pages, {
            base: config.build.temp,
            cwd: config.build.temp,
        })
            >处理 ref链接
            .pipe(plugins.useref({ searchPath: [config.build.temp, "."] }))
            >压缩js，css，html
            .pipe(plugins.if(/\.js$/, plugins.uglify()))
            .pipe(plugins.if(/\.css$/, plugins.cleanCss()))
            .pipe(
                plugins.if(
                    /\.html$/,
                    plugins.htmlmin({
                        collapseWhitespace: true,
                        minifyCSS: true,
                        minifyJS: true,
                    })
                )
            )
            .pipe(dest(config.build.dist))
    );
};
```

> 组合任务 文件
> `const compile = parallel(scripts, styles, pages)`;
> 组合任务 上线

```
const build = series(
    delText,
    parallel(series(compile, useref), images, fonts, extra)
);
```

> 组合任务 开发

`const dev = series(delText, build, server);`

```
module.exports = {
    clean: delText,
    build,
    dev,
};
```
