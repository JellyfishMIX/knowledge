# the difference between npm run buid and yarn run build

`yarn build` and `npm build` are not existing commands by default. I think you mean `yarn run build` or `npm run build`.

`build` is a command which can be specified in your `package.json` file on the `scripts` property. See the example below.

```
{
    "name": "mypackage",
    "version": "0.1.0",
    "scripts": {
       "build": "webpack --config webpack.dev.js"
    }
}
```

In this example, `build` is a shortcut for launching command `webpack --config webpack.dev.js`. You can use every keyword you want to define some shortcuts to launch commands.

And the only difference between the two commands it's the JS dependency manager you're using, yarn or npm.

More infos :

https://yarnpkg.com/lang/en/

https://www.npmjs.com/



## Reference/Quote

[jtouzy - stackoverflow](https://stackoverflow.com/questions/54693223/what-does-yarn-build-command-do-are-npm-build-and-yarn-build-similar)