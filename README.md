### Nodejs Cheat Sheet [https://ratu.dev/snippets/nodejs-cheatsheet](https://ratu.dev/snippets/nodejs-cheatsheet)

## File-System Essentials (icon: Folder)

### Files CRUD (icon: Files)

```js
import fs from 'node:fs/promises'

await fs.readFile('file.txt', 'utf8')
await fs.writeFile('file.txt', 'data')
await fs.appendFile('file.txt', 'more')
await fs.copyFile('src.txt', 'dest.txt')
await fs.rename('old.txt', 'new.txt')
await fs.rm('file.txt')
```

[Nodejs Docs](https://nodejs.org/api/fs.html)

<summary>CRUD operations for files - create, read, update, delete, as well as copy, rename.</summary>

### Directories CRUD

```js
import fs from 'node:fs/promises'

await fs.mkdir('dir', {recursive: true})
await fs.rmdir('dir')
await fs.rm('dir', {recursive: true})
await fs.readdir('dir') // return files
await fs.cp('src/', 'dest/', {recursive: true}) // copy
```

<summary>
    CRUD operations for directories - create, remove, list contents, and copy directories with recursive options.
</summary>

### Directory walk

```js
import {readdir} from 'node:fs/promises'
import {join} from 'node:path'

async function* walk(dir) {
    for (const entry of await readdir(dir, {withFileTypes: true})) {
        const res = join(dir, entry.name)
        if (entry.isDirectory()) yield* walk(res)
        else yield res
    }
}

for await (const file of walk('src')) console.log(file)
```

<summary>
    Recursively traverse directory structure using async generators to process all files in subdirectories.
</summary>

### Glob

```js
import fg from 'fast-glob' // external dependency

const files = await fg('**/*.js', {cwd: 'src'})
```

<summary>
    Pattern-based file matching using fast-glob library to find files matching specific patterns across directories.
</summary>

### Large-file streams `createReadStream`

```js
import {createReadStream} from 'node:fs'
import readline from 'node:readline'

const rl = readline.createInterface({input: createReadStream('large.log')})
for await (const line of rl) {
    // do something with each line
}
```

<summary>
    Memory-efficient processing of large files by streaming data line-by-line instead of loading entire file into
    memory.
</summary>

### GZIP file

```js
import {createReadStream, createWriteStream} from 'node:fs'
import {createGzip} from 'node:zlib'

createReadStream('data.json').pipe(createGzip()).pipe(createWriteStream('data.json.gz'))
```

```js
import archiver from 'archiver'
import {createWriteStream} from 'node:fs'

const output = createWriteStream('site.zip')
const archive = archiver('zip', {zlib: {level: 9}})

archive.pipe(output)
archive.directory('public/', false)
await archive.finalize()
```

<summary>
    File compression using Node.js built-in zlib for GZIP and archiver library for ZIP archives with streaming support.
</summary>

### unzip archives

```js
import fs from 'node:fs'
import unzipper from 'unzipper'

export function unzip(archivePath: string, dest = './out'): Promise<void> {
    return fs
        .createReadStream(archivePath)
        .pipe(unzipper.Extract({path: dest}))
        .promise() // unzipper adds a neat promise helper
}
```

<summary>
    Extract ZIP archives using unzipper library with streaming support and promise-based API for clean async handling.
</summary>

### ZIP

```js
import {zip} from 'zip-a-folder'

export async function zipDir(src: string, out = 'archive.zip') {
    await zip(src, out)
}
```

<summary>
    Simple directory compression using zip-a-folder library for basic ZIP archive creation with minimal configuration.
</summary>

### ZIP with exclude:

```js
import fs from 'node:fs'
import archiver from 'archiver'

export async function zipDir(src: string, out = 'archive.zip'): Promise<void> {
    return new Promise((resolve, reject) => {
        const output = fs.createWriteStream(out)
        const archive = archiver('zip', {zlib: {level: 9}})

        output.on('close', resolve)
        archive.on('error', reject)
        archive.pipe(output)

        // include only .js/.ts/.json, skip maps, dot-files, node_modules
        archive.glob('**/*.{js,ts,json}', {
            cwd: src,
            ignore: ['node_modules/**', '**/*.map', '**/.*']
        })

        archive.finalize()
    })
}
```

<summary>
    Advanced ZIP creation with selective file inclusion using glob patterns and exclusion rules to filter specific file
    types and directories.
</summary>

## CLI Parameters & Environment (icon: Terminal)

### Raw `process.argv` parsing

```js
// node index.js deploy
process.argv
// [0] node [1] index.js [2] deploy

const [, , ...args] = process.argv // or  process.argv.slice(2)
// args: [0] deploy
```

<summary>
    Basic command-line argument parsing using Node.js built-in process.argv to extract command-line parameters passed to
    the script.
</summary>

### CLI (no libs)

```js
// @returns {{ [key: string]: string|boolean }}
function parseArgs(argv) {
    const result = {}
    let currentKey = null

    argv.forEach(token => {
        if (token.startsWith('--')) {
            // Long flag or key
            currentKey = token.slice(2)
            result[currentKey] = true
        } else if (token.startsWith('-')) {
            // Short flag or key
            currentKey = token.slice(1)
            result[currentKey] = true
        } else if (currentKey) {
            // Assign value to the last key
            result[currentKey] = token
            currentKey = null
        }
    })

    return result
}

parseArgs(process.argv.slice(2))
```

<summary>
    Custom command-line argument parser that handles both long (--flag) and short (-f) flags with values, without
    external dependencies.
</summary>

### CLI `yargs`

```js
import yargs from 'yargs'

const argv = yargs
    .command('deploy', 'Deploy the app', y => y.option('env', {type: 'string', default: 'dev'}))
    .help().argv

console.log(argv.env)
```

<summary>
    Command-line interface builder using yargs library for creating complex CLI applications with commands, options, and
    built-in help.
</summary>

### CLI `commander`

```js
import {Command} from 'commander'

const program = new Command()

program
    .name('mycli')
    .description('Build or deploy your project')
    .version('1.0.0')
    .argument('<action>', 'what to do: build or deploy')
    .option('-w, --watch', 'rebuild on file changes (only for build)')
    .action((action, opts) => {
        // ...
    })

program.parse(process.argv)
```

<summary>
    Modern CLI framework using commander.js for building command-line applications with subcommands, arguments, options,
    and automatic help generation.
</summary>

### Interactive prompts (`inquirer`, `prompts`)

```js
import prompts from 'prompts'

const response = await prompts({
    type: 'select',
    name: 'region',
    message: 'Pick a region',
    choices: [
        {title: 'US East', value: 'us-east1'},
        {title: 'Europe', value: 'europe-west1'}
    ]
})
console.log(response.region)
```

<summary>
    Interactive command-line prompts using prompts library for user input with various input types like select, text,
    confirm, and multi-select.
</summary>

-   ENV access with `process.env`

```js
const apiKey = process.env.API_KEY
```

### `dotenv` - Env Variables

`import dotenv from 'dotenv' - dotenv()`

```bash
# .env
API_KEY=abcd1234
DEBUG=true
```

-   `cross-env`

```jsonc
// package.json scripts
"scripts": {
  "start": "cross-env NODE_ENV=production node index.js"
}
```

<summary>
    Environment variable management using dotenv for loading .env files and cross-env for cross-platform environment
    variable setting in npm scripts.
</summary>

---

## Process & Child Processes (icon: GitFork)

### `spawn`

```js
import {spawn} from 'child_process'

spawn('yarn', ['audit', '--json']).stdout.on('data', rawLine => {
    if (rawLine) {
        const item = JSON.parse(line)
        if (item?.type === 'auditSummary' && item.data?.vulnerabilities?.critical > 0) {
            console.error('Found critical vulnerability')
            process.exit(1)
        }
    }
})
```

<summary>
    Asynchronous child process execution using spawn for streaming output and real-time data processing with event-based
    communication.
</summary>

### `exec`

```js
import {execSync} from 'child_process'
execSync('yarn audit --json', {encoding: 'utf-8'})
```

-   Piping `stdout` / `stderr`
-   Signal handling (`SIGINT`, `SIGTERM`)
    -   `SIGINT`: the “interrupt” signal (Ctrl+C in your terminal)
    -   `SIGTERM`: the “terminate” signal (default kill command)
-   `process.cwd()`

<summary>
    Synchronous child process execution using execSync for simple command execution with buffered output and blocking
    behavior.
</summary>

### Exit Codes

```js
process.exit(1)
```

-   `1` - Success ✅ - to help memorize, think \*"1 finger up 👍"
-   `0` - Failure ❌ - to help memorize, think \*“0 trophies 🏆”

### Signals

-   `SIGINT`: the “interrupt” signal (Ctrl+C in your terminal)
-   `SIGTERM`: the “terminate” signal (default kill command)

```js
process.on('SIGINT', () => {
    console.log('Caught SIGINT (Ctrl+C). Cleaning up...')
    process.exit(0)
})

process.on('SIGTERM', () => {
    console.log('Caught SIGTERM (kill). Cleaning up...')
    process.exit(0)
})
```

<summary>
    Process signal handling for graceful shutdown using SIGINT (Ctrl+C) and SIGTERM (kill) with cleanup procedures.
</summary>

## HTTP Client Requests (icon: Globe)

### Native `fetch` Post

```js
const res = await fetch(url)
const json = await res.json()

// POST JSON
await fetch(url, {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({foo: 'bar'})
})
```

<summary>
    Native fetch API for HTTP requests with JSON parsing and POST requests with proper headers and body serialization.
</summary>

### Native `fetch` GET

```js
const res = await fetch('https://api.example.com/data')
const data = await res.json()
```

<summary>Simple GET requests using native fetch API with JSON response parsing for API data retrieval.</summary>

### Cancel (Abort) fetch

```js
const controller = new AbortController()
const {signal} = controller

// 2. Pass the signal to fetch
fetch('https://api.example.com/data', {signal})

controller.abort()
```

<summary>
    Request cancellation using AbortController to cancel ongoing fetch requests with signal-based abort mechanism.
</summary>

### Streaming downloads

```js
import fs from 'fs'
const res = await fetch(fileUrl)
const dest = fs.createWriteStream('file.zip')
await new Promise((r, e) => res.body.pipe(dest).on('finish', r).on('error', e))
```

<summary>
    Memory-efficient file downloads using streaming to pipe response body directly to file system without loading entire
    file into memory.
</summary>

### Fetch with retry

```js
async function fetchWithRetry(url, options, retries = 3, delay = 1000) {
    try {
        return fetch(url, options)
    } catch (err) {
        if (retries === 0) throw err
        await new Promise(r => setTimeout(r, delay))
        return fetchWithRetry(url, options, retries - 1, delay * 2)
    }
}
```

<summary>
    Resilient HTTP requests with exponential backoff retry logic for handling temporary network failures and transient
    errors.
</summary>

## TypeScript in Node (icon: FileCode2)

### Essential `tsconfig` flags

```jsonc
{
    "compilerOptions": {
        "target": "ES2020", // modern JS
        "module": "CommonJS", // or "ESNext" for ESM
        "strict": true, // all strict type-checks
        "esModuleInterop": true, // default imports from CJS
        "skipLibCheck": true, // faster builds
        "outDir": "dist", // compiled output
        "rootDir": "src", // source files
        "moduleResolution": "node" // resolve like Node.js
    }
}
```

<summary>
    Core TypeScript configuration options for modern JavaScript targets, strict type checking, and proper module
    resolution in Node.js environment.
</summary>

### Live reload `ts-node`

```bash
# install once
npm install -D ts-node typescript

# run with auto-recompile on file change
npx ts-node-dev src/index.ts
```

<summary>
    Development workflow using ts-node for direct TypeScript execution with automatic recompilation and file watching
    for rapid development.
</summary>

### Live reload `tsx --watch`

```bash
npm install -D tsx

# watch mode
npx tsx --watch src/index.ts
```

<summary>
    Fast TypeScript execution using tsx with watch mode for zero-config development with esbuild-based compilation.
</summary>

### Build → run `dist/` output

```bash
npx tsc

node dist/index.js
```

<summary>
    Production build workflow using TypeScript compiler to generate JavaScript output in dist directory for deployment.
</summary>

### ES-module resolution in TS

```json
{
    "compilerOptions": {
        "module": "ESNext",
        "moduleResolution": "NodeNext"
    }
}
```

```jsonc
"type": "module"
```

-   `@types/node` for built-ins

<summary>
    ES module configuration for TypeScript with NodeNext module resolution and package.json type field for modern ES
    module support.
</summary>

## Automation & Scheduling (icon: Clock)

### `node-cron` schedules

```js
import cron from 'node-cron'
//            ┌────────────── second (optional)
//            │ ┌──────────── minute
//            │ │ ┌────────── hour
//            │ │ │ ┌──────── day of month
//            │ │ │ │ ┌────── month
//            │ │ │ │ │ ┌──── day of week
//            │ │ │ │ │ │
//            │ │ │ │ │ │
//            * * * * * *
cron.schedule('* * * * * *', () => {
    console.log('running a task every minute')
})
```

<summary>
    Scheduled task execution using node-cron with cron expression syntax for time-based automation and recurring jobs.
</summary>

## Logging, Debugging, Safety Nets (icon: Bug)

### Console vs structured logs (`Pino`)

```js
import pino from 'pino'
const logger = pino({level: process.env.LOG_LEVEL || 'info'})
logger.info({user: 'alice'}, 'User logged in')
```

<summary>
    Structured logging with Pino for production-ready logging with configurable log levels and structured data output.
</summary>

### Colored output (`chalk`)

```js
import chalk from 'chalk'
console.log(chalk.green('✅ OK'), 'Server started on port', chalk.blue(port))
```

<summary>
    Terminal colorization using chalk library for enhanced CLI output with colored text, emojis, and visual feedback.
</summary>

### Inspector flag (`--inspect`)

`node --inspect=127.0.0.1:9229 script.js`

Or `launch.json` in VSCode

```jsonc
{
    "type": "node",
    "request": "attach",
    "name": "Attach to Script",
    "address": "localhost",
    "port": 9229
}
```

<summary>
    Node.js debugging setup using --inspect flag for Chrome DevTools integration and VSCode debugging configuration.
</summary>

## Packaging & Distribution (icon: Package)

### Publish CLI to `npm`

```jsonc
{
    "name": "my-cli",
    "version": "1.0.0",
    "bin": {
        "my-cli": "./index.js"
    }
}
```

```bash
npm login
npm publish --access public
```

<summary>
    CLI tool publishing to npm registry with package.json bin field for global installation and npm publish workflow.
</summary>

### Semver & changelog basics

```bash
npm version patch   # 1.0.0 → 1.0.1
npm version minor   # 1.0.1 → 1.1.0
npm version major   # 1.1.0 → 2.0.0
```

<summary>
    Semantic versioning using npm version commands for automated version bumping and release management following semver
    conventions.
</summary>

---

## Streams & Buffers (icon: Waves)

### `Readable` stream

```js
import {createReadStream} from 'fs'
const r = createReadStream('./input.txt', {encoding: 'utf8', highWaterMark: 64 * 1024})
r.on('data', chunk => console.log('got %d bytes', chunk.length))
r.on('end', () => console.log('finished'))
// Or: let buf; while (null !== (buf = r.read())) { … }
```

<summary>
    Readable streams for data consumption with event-based processing, configurable buffer sizes, and chunk-by-chunk
    data handling.
</summary>

### `Writable` stream

```js
import {createWriteStream} from 'fs'
const w = createWriteStream('./output.txt', {flags: 'w'})
w.write('Hello') // returns false if buffer is full
w.end('World', () => console.log('done'))
w.on('drain', () => console.log('ready for more'))
```

<summary>
    Writable streams for data output with backpressure handling, buffer management, and completion callbacks for data
    writing operations.
</summary>

### `Pipe` Chains (Simple chaining)

```js
import {createReadStream, createWriteStream} from 'fs'
createReadStream('big.log').pipe(createWriteStream('copy.log'))
```

<summary>
    Simple stream piping for direct data transfer from readable to writable streams with automatic backpressure
    handling.
</summary>

### `Pipe` chains (Transform)

```js
import zlib from 'zlib'
createReadStream('app.log')
    .pipe(zlib.createGzip()) // transform
    .pipe(createWriteStream('app.log.gz'))
```

<summary>
    Transform stream piping for data processing with intermediate transformation steps like compression, filtering, or
    data conversion.
</summary>

### Custom `Transform`

```js
import {Transform} from 'stream'
class Uppercase extends Transform {
    _transform(chunk, _, callback) {
        this.push(chunk.toString().toUpperCase())
        callback()
    }
}
// Usage:
process.stdin.pipe(new Uppercase()).pipe(process.stdout)
```

<summary>
    Custom transform stream implementation for data processing with _transform method to modify data as it flows through
    the stream pipeline.
</summary>

### Real-World Example: Compress HTTP response with back-pressure control

```js
import http from 'http'
import zlib from 'zlib'

http.createServer((req, res) => {
    const src = createReadStream('./large.json')
    res.writeHead(200, {'Content-Encoding': 'gzip'})
    const gzip = zlib.createGzip({highWaterMark: 32 * 1024})

    src.pipe(gzip) // transform + back-pressure
        .pipe(res, {end: true})
}).listen(3000)
```

<summary>
    HTTP server with streaming compression using transform streams and backpressure control for efficient large file
    serving with gzip compression.
</summary>

### `String` -> `Buffer`

```js
const buf = Buffer.from('✓ Success', 'utf8') // buffer to string is toString()
```

<summary>
    Buffer creation from strings with UTF-8 encoding for binary data handling and string-to-buffer conversion
    operations.
</summary>

## Event Loop & Async Patterns (icon: RefreshCcw)

### Micro-tasks/Macro-tasks

```js
setTimeout(() => console.log('timeout'), 0) // macro-task (timers)
setImmediate(() => console.log('immediate')) // macro-task (check phase)
process.nextTick(() => console.log('nextTick')) // micro-task

Promise.resolve().then(() => console.log('promise')) // micro-task

console.log('end')
// Likely output:
// start
// end
// nextTick
// promise
// timeout      (or immediate first on some versions)
// immediate
```

<summary>
    Node.js event loop execution order with micro-tasks (nextTick, promises) and macro-tasks (setTimeout, setImmediate)
    demonstrating task prioritization.
</summary>

### `timers` helpers

```js
import {setTimeout as delay, setImmediate} from 'timers/promises'

// Delay for 500ms
await delay(500)
```

Or without `timers` helpers:

```js
await new Promise(resolve => setTimeout(resolve, 500))
```

<summary>
    Promise-based timer utilities using timers/promises for cleaner async/await syntax and alternative Promise-based
    delay implementation.
</summary>

### Graceful shutdown

```js
process.on('SIGTERM', async () => {
    console.log('Shutting down...')
    server.close() // stop accepting new connections
    // wait up to 5s for existing requests
    await wait(5000)
    console.log('Done')
    process.exit(0)
})
```

<summary>
    Graceful application shutdown handling with server cleanup, connection draining, and proper process termination for
    production deployments.
</summary>

## Testing & Coverage (icon: FlaskConical)

### Nodejs asserts/tests

-   Run test: `node --test`

```js
import {test} from 'node:test'
import assert from 'node:assert'

const transform = data => {
    assert.strictEqual(typeof data, 'string')
}
```

<summary>
    Built-in Node.js testing framework using node:test and node:assert modules for writing and running tests without
    external dependencies.
</summary>

### Sample test:

```js
import {test} from 'node:test'
import assert from 'node:assert/strict'

export function capitalize(str) {
    if (typeof str !== 'string') throw new TypeError('Expected a string')
    return str.charAt(0).toUpperCase() + str.slice(1)
}

test('capitalize uppercases first letter', () => {
    assert.strictEqual(capitalize('hello'), 'Hello')
})

test('capitalize throws on non-string', () => {
    assert.throws(() => capitalize(123), {
        name: 'TypeError',
        message: 'Expected a string'
    })
})
```

<summary>
    Complete test example demonstrating function testing with positive and negative test cases, error handling
    validation, and strict assertion usage.
</summary>

### Asserts:

```js
assert.deepStrictEqual(cfg, {port: 3000})
assert.doesNotMatch('I will fail', /fail/, 'they match, sorry!')
assert.fail('boom')
assert.ifError(null) // throw if value is not null/undefined
assert.match('I will pass', /pass/)
assert.ok(typeof '123' === 'string')
```

<summary>
    Common assertion methods for testing including deep equality, pattern matching, error checking, and boolean
    validation with descriptive error messages.
</summary>

---

## Live Reload & Watchers (icon: RefreshCw)

### `nodemon`

```bash
npx nodemon index.js
```

```json
{
    "watch": ["src"],
    "ext": "js,ts,json",
    "ignore": ["src/**/*.spec.ts"],
    "exec": "node ./src/index.js"
}
```

Docs: [https://github.com/remy/nodemon#nodemon](https://github.com/remy/nodemon#nodemon)

<summary>
    Development tool for automatic application restart on file changes with configurable file watching, extensions, and
    execution commands.
</summary>

### `tsx --watch`

```bash
npx tsx watch src/index.ts
```

Why use it: zero-config, fast startup for both TS and ESM.

Docs: [https://github.com/esbuild-kit/tsx#readme](https://github.com/esbuild-kit/tsx#readme)

<summary>
    Fast TypeScript execution with watch mode using esbuild for zero-config development with automatic file watching and
    restart.
</summary>

### File watchers with `chokidar`

```js
import {spawn} from 'child_process'
import chokidar from 'chokidar'

let proc
const start = () => {
    if (proc) proc.kill()
    proc = spawn('node', ['./src/index.js'], {stdio: 'inherit'})
}

chokidar.watch('./src/**/*.js').on('all', (evt, path) => {
    console.log(`${evt} detected on ${path}. Restarting…`)
    start()
})

start()
```

<summary>
    Custom file watching implementation using chokidar for process restart on file changes with child process management
    and event handling.
</summary>

### Hot reload for ESM loaders

```bash
node --watch ./src/index.mjs
```

<summary>
    Native Node.js file watching for ES modules using --watch flag for automatic restart on file changes without
    external dependencies.
</summary>

### Production process manager

```bash
pm2 start app.js
```

https://www.npmjs.com/package/pm2

<summary>
    Production process management using PM2 for application deployment with process monitoring, restart capabilities,
    and cluster mode support.
</summary>

---

## Concurrency (icon: Cpu)

### Spawning `worker_threads`

```js
import {Worker} from 'worker_threads'

function runService(workerData) {
    return new Promise((resolve, reject) => {
        const worker = new Worker(new URL('./worker.js', import.meta.url), {workerData})
        worker
            .on('message', resolve)
            .on('error', reject)
            .on('exit', code => {
                if (code !== 0) reject(new Error(`Worker stopped with exit code ${code}`))
            })
    })
}

const result = await runService({fib: 40})
console.log(`Fibonacci result: ${result}`)
```

```js
// worker.js
import {parentPort, workerData} from 'worker_threads'

function fib(n) {
    return n < 2 ? n : fib(n - 1) + fib(n - 2)
}

parentPort.postMessage(fib(workerData.fib))
```

<summary>
    Worker thread implementation for CPU-intensive tasks with message passing between main thread and worker, error
    handling, and promise-based communication.
</summary>

### Threads with `piscina`

```js
// install: npm install piscina
import Piscina from 'piscina'
import {resolve} from 'path'

const piscina = new Piscina({
    filename: resolve(__dirname, 'task.js'),
    maxThreads: 4
})

;(async () => {
    const results = await Promise.all([piscina.run({file: 'large.csv'}), piscina.run({file: 'report.json'})])
    console.log(results)
})()
```

<summary>
    Thread pool management using Piscina for efficient parallel task execution with configurable thread limits and
    promise-based task submission.
</summary>

### `MessagePort` & `Transferables`

```js
// parent.js
import {MessageChannel} from 'worker_threads'

const {port1, port2} = new MessageChannel()
const worker = new Worker(new URL('./worker.js', import.meta.url), {
    transferList: [port2],
    workerData: {port: port2}
})

port1.on('message', data => {
    console.log('Received buffer length:', data.byteLength)
})

const buf = new ArrayBuffer(1024 * 1024)
port1.postMessage(buf, [buf]) // zero-copy transfer
```

```js
// worker.js
import {parentPort, workerData} from 'worker_threads'

const port = workerData.port
parentPort.unref() // allow process to exit if only this port is left

port.on('message', buf => {
    // process the ArrayBuffer directly
    port.postMessage(buf, [buf])
})
```

<summary>
    Advanced worker communication using MessagePort and transferable objects for zero-copy data transfer and efficient
    memory management between threads.
</summary>

### Cluster - example

```js
// cluster-server.js
import cluster from 'cluster'
import http from 'http'
import os from 'os'

if (cluster.isPrimary) {
    const cpus = os.cpus().length
    for (let i = 0; i < cpus; i++) cluster.fork()
    cluster.on('exit', worker => console.log(`Worker ${worker.process.pid} died, spawning a new one`))
} else {
    http.createServer((req, res) => {
        // simulate work
        res.end(`Handled by ${process.pid}`)
    }).listen(8000, () => console.log(`Worker ${process.pid} listening`))
}
```

<summary>
    Multi-process clustering using Node.js cluster module for load balancing across CPU cores with automatic worker
    management and fault tolerance.
</summary>

---

## Security & HTTPS Basics (icon: Lock)

### HTTPS server & TLS certs

```js
import https from 'https'
import fs from 'fs'
import express from 'express'

const app = express()
// Load key & cert (e.g. from Let's Encrypt)
const options = {
    key: fs.readFileSync('/etc/letsencrypt/live/example.com/privkey.pem'),
    cert: fs.readFileSync('/etc/letsencrypt/live/example.com/fullchain.pem')
}

https.createServer(options, app).listen(443, () => console.log('HTTPS running on port 443'))
```

<summary>
    Secure HTTPS server setup with TLS certificates from Let's Encrypt for encrypted communication and production-ready
    SSL/TLS configuration.
</summary>

## CLI / UX Enhancements (icon: Command)

### Colors (`chalk`)

```js
console.log(chalk.green('✔ Build succeeded!'))
console.log(chalk.yellow('⚠ Deprecation warning:'))
console.log(chalk.red('✖ Build failed'))
```

<summary>
    Enhanced CLI output using chalk for colored text, emojis, and visual status indicators to improve user experience
    and readability.
</summary>

### Spinners (`ora`)

```js
import ora from 'ora'

const spinner = ora('Downloading assets...').start()
await downloadAssets()
spinner.succeed('Assets downloaded')
```

<summary>
    Loading indicators using ora for animated spinners during long-running operations with success/error state
    management.
</summary>

### Progress bars (`cli-progress`)

```js
import {SingleBar, Presets} from 'cli-progress'

const bar = new SingleBar({format: 'Progress |{bar}| {percentage}%'}, Presets.shades_classic)
const total = files.length
bar.start(total, 0)

for (const [i, file] of files.entries()) {
    await upload(file)
    bar.update(i + 1)
}
bar.stop()
```

<summary>
    Progress tracking using cli-progress for visual progress bars with customizable formatting and real-time progress
    updates.
</summary>

### Tables (`cli-table`)

```js
import Table from 'cli-table'

const table = new Table({head: ['Name', 'Status', 'Time(ms)']})
results.forEach(r => table.push([r.name, r.ok ? '✔' : '✖', r.time]))
console.log(table.toString())
```

<summary>
    Tabular data display using cli-table for structured output with headers, rows, and formatted table rendering in
    terminal.
</summary>

---

## Node Runtime Flags & Profiling (icon: Flag)

### `--inspect` / `--inspect-brk`

```bash
node --inspect-brk my-script.js
# In Chrome: open chrome://inspect → “Open dedicated DevTools”
```

### `NODE_OPTIONS` env tweaks

```bash
export NODE_OPTIONS="--trace-deprecation --inspect=9229"
npm test  # now runs with both flags automatically
```

<summary>
    Environment variable for setting Node.js runtime options globally across all npm scripts and node commands.
</summary>

---

## Containerization & Serverless Deploy (icon: Server)

### Minimal Node `Dockerfile`

```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build   # e.g., transpile TS or bundle

# Production stage
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --omit=dev
CMD ["node", "dist/index.js"]
```

<summary>
    Multi-stage Docker build for Node.js applications with optimized production images and development dependencies
    excluded.
</summary>

### Docker Compose

```yaml
version: '3.8'
services:
    app:
        build: .
        volumes:
            - .:/app # live sync code
            - /app/node_modules
        ports:
            - '3000:3000'
        environment:
            NODE_ENV: development
        command: npm run dev # e.g., nodemon
```

<summary>
    Docker Compose configuration for development environment with volume mounting, port mapping, and live code
    synchronization.
</summary>

### AWS Lambda handler basics

```js
exports.handler = async event => {
    console.log('Received:', event)
    return {
        statusCode: 200,
        body: JSON.stringify({message: 'Hello, Lambda!'})
    }
}
```

-   Docs: [https://docs.aws.amazon.com/lambda/latest/dg/nodejs-handler.html](https://docs.aws.amazon.com/lambda/latest/dg/nodejs-handler.html)

```bash
aws lambda invoke \
  --function-name my-function \
  --payload '{}' response.json
```

<summary>
    Serverless function implementation for AWS Lambda with event-driven architecture and HTTP response formatting for
    API Gateway integration.
</summary>

### Cloudflare Workers

```js
export default {
    async fetch(request) {
        return new Response('Hello from the Edge!')
    }
}
```

```bash
npm install -g wrangler
wrangler init my-worker
```

-   Docs: [https://developers.cloudflare.com/workers/](https://developers.cloudflare.com/workers/)

<summary>
    Edge computing with Cloudflare Workers using fetch API for serverless functions deployed globally with Wrangler CLI
    tooling.
</summary>

### Vercel

```js
// api/hello.js
export default async function handler(req) {
    return new Response(JSON.stringify({msg: 'Hi from Vercel Edge'}), {
        headers: {'Content-Type': 'application/json'}
    })
}
```

```jsonc
// vercel.json
{
    "functions": {"api/*.js": {"runtime": "edge"}}
}
```

-   Docs: [https://vercel.com/docs/functions](https://vercel.com/docs/functions)

<summary>
    Serverless deployment on Vercel platform with edge runtime configuration and API route handling for modern web
    applications.
</summary>

## Boot & Run Basics (icon: PlayCircle)

### Shebang + `chmod` executable

```bash
echo '#!/usr/bin/env node' > script.mjs && chmod +x script.mjs
```

```js
#!/usr/bin/env node
console.log(2)
```

```bash
./script.mjs
```

-   Article: [Node.js shebang](https://alexewerlof.medium.com/node-shebang-e1d4b02f731d)

<summary>
    Executable Node.js scripts using shebang directive and chmod permissions for direct script execution without node
    command prefix.
</summary>
