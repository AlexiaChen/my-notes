
主要是执行 `truffle compile` 报错，说是sloc 这个solidity的编译器下载不下来

```txt
Compiling your contracts...
===========================
✖ Fetching solc version list from solc-bin. Attempt #1
✖ Fetching solc version list from solc-bin. Attempt #2
✖ Fetching solc version list from solc-bin. Attempt #3
✖ Fetching solc version list from solc-bin. Attempt #4
✖ Fetching solc version list from solc-bin. Attempt #5
✖ Fetching solc version list from solc-bin. Attempt #6
Error: Failed to fetch the Solidity compiler from the following locations: https://relay.trufflesuite.com/solc/emscripten-wasm32/,https://binaries.soliditylang.org/emscripten-wasm32/,https://relay.trufflesuite.com/solc/emscripten-asmjs/,https://binaries.soliditylang.org/emscripten-asmjs/,https://solc-bin.ethereum.org/bin/,https://ethereum.github.io/solc-bin/bin/. Are you connected to the internet?


    at VersionRange.<anonymous> (/home/mathxh/.nvm/versions/node/v18.12.0/lib/node_modules/truffle/build/webpack:/packages/compile-solidity/dist/compilerSupplier/loadingStrategies/VersionRange.js:183:1)
    at Generator.next (<anonymous>)
    at /home/mathxh/.nvm/versions/node/v18.12.0/lib/node_modules/truffle/build/webpack:/packages/compile-solidity/dist/compilerSupplier/loadingStrategies/VersionRange.js:8:1
    at new Promise (<anonymous>)
    at exports.modules.780.__awaiter (/home/mathxh/.nvm/versions/node/v18.12.0/lib/node_modules/truffle/build/webpack:/packages/compile-solidity/dist/compilerSupplier/loadingStrategies/VersionRange.js:4:1)
    at VersionRange.getSolcFromCacheOrUrl (/home/mathxh/.nvm/versions/node/v18.12.0/lib/node_modules/truffle/build/webpack:/packages/compile-solidity/dist/compilerSupplier/loadingStrategies/VersionRange.js:174:1)
    at VersionRange.<anonymous> (/home/mathxh/.nvm/versions/node/v18.12.0/lib/node_modules/truffle/build/webpack:/packages/compile-solidity/dist/compilerSupplier/loadingStrategies/VersionRange.js:206:1)
    at Generator.throw (<anonymous>)
    at rejected (/home/mathxh/.nvm/versions/node/v18.12.0/lib/node_modules/truffle/build/webpack:/packages/compile-solidity/dist/compilerSupplier/loadingStrategies/VersionRange.js:6:42)
    at processTicksAndRejections (node:internal/process/task_queues:96:5)
Truffle v5.6.2 (core: 5.6.2)
Node v16.16.0
```


说是不是网络的原因，因为我也开了代理，查了SO [node.js - Error: Failed to fetch the Solidity compiler - Stack Overflow](https://stackoverflow.com/questions/73355583/error-failed-to-fetch-the-solidity-compiler) 上说是要`sudo truffle compile`，  随后编辑`/etc/sudoers` 文件中的secure_path，添加which truffle出来的路径，用 `sudo -E truffle compile` 来保留当前用户的环境便令去执行命令 [linux - How to keep environment variables when using sudo - Stack Overflow](https://stackoverflow.com/questions/8633461/how-to-keep-environment-variables-when-using-sudo)

如果实在不行，看看配置的sloc的版本对不对，好像版本号不存在也会报这样的错误。