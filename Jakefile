/**
 * Instructions for updating this file.
 * - this file must be updated everytime a new cross-folder dependency is added
 *   into any of the [refs.ts] file. For instance, if suddenly you add a
 *   reference to [../storage/whatever.ts] in [build/libwab.ts], then you must
 *   update the dependencies of the [build/libwab.d.ts] task.
 **/
var assert = require('assert');
var child_process = require("child_process");
var fs = require("fs");
var path = require("path");
var source_map = require("source-map");
var events = require("events");

events.EventEmitter.defaultMaxListeners = 32;

var head;
function getGitHead() {
  if (!head)
    head = child_process.execSync("git rev-parse HEAD");
  return head;
}

jake.addListener("start", function () {
  if (!fs.existsSync("build"))
    fs.mkdirSync("build");
});

var branchingFactor = 32;

// The list of files generated by the build.
var generated = [
  'shell/npm/package.json',
  'shell/npm/bin/touchdevelop.js',
  'results.html',
  'results.json',
];

// A list of targets we compile with the --noImplicitAny flag.
var noImplicitAny = {
  "build/browser.d.ts": null,
};


// On Windows, merely changing a file in the directory does *not* change that
// directory's mtime, meaning that we can't depend on a directory, but rather
// need to depend on all the files in a directory. [expand] takes a list of
// dependencies, and for filenames that seem to be directories, performs a
// manual expansion.
function expand(dependencies) {
  var isFolder = function (f) {
    return f.indexOf(".") < 0 && f.indexOf("/") < 0;
  };
  var acc = [];
  dependencies.forEach(function (f) {
    if (isFolder(f)) {
      var l = new jake.FileList();
      l.include(f+"/*");
      acc = acc.concat(l.toArray());
    } else {
      acc.push(f);
    }
  });
  return acc;
}


// This function tries to be "smart" about the target.
// - if the target is of the form "foobar/refs.ts", the output is bundled with
//   [--out] into "build/foobar.js", and "build/foobar.d.ts" is also generated;
// - otherwise, the file is just compiled as a single file into the "build/"
//   directory
function mkSimpleTask(production, dependencies, target) {
    var sourceRoot = (process.env.TRAVIS
      ? "https://github.com/Microsoft/TouchDevelop/raw/"+getGitHead()+"/"
      : "/editor/local/");
    var tscCall = [
      "node node_modules/typescript/bin/tsc",
      "--noEmitOnError",
      "--removeComments",
      "--target ES5",
      "--module commonjs",
      "--declaration", // not always strictly needed, but better be safe than sorry
    ];
    if (process.env.TD_SOURCE_MAPS) {
      tscCall.push("--sourceMap");
      tscCall.push("--sourceRoot "+sourceRoot);
    }
    if (production in noImplicitAny)
      tscCall.push("--noImplicitAny");
    var match = target.match(/(\w+)\/refs.ts/);
    if (match) {
      tscCall.push("--out build/"+match[1]+".js");
    } else {
      tscCall.push("--outDir build/");
    }
    tscCall.push(target);
    return file(production, expand(dependencies), { async: true, parallelLimit: branchingFactor }, function () {
      var task = this;
      console.log("[B] "+production);
      jake.exec(tscCall.join(" "), { printStdout: true }, function () {
          task.complete();
      });
    });
}

// runs madoko and generates the html file
function mdkTask(production, dependencies, target) {
    return file(production, expand(dependencies), { async: true, parallelLimit: branchingFactor }, function () {
      var task = this;
	  jake.mkdirP('build/portal')
      console.log("[M] "+production);
      jake.exec("node node_modules/madoko/lib/cli.js --verbose --odir=build/portal " + target, { printStdout: true }, function () {
          task.complete();
      });
    });
}

// A series of compile-and-run rules that generate various files for the build
// system.
function runAndComplete(cmds, task) {
    jake.exec(cmds, {}, function() {
        task.complete();
    });
}

mkSimpleTask('build/genmeta.js', [ 'tools', ], "tools/genmeta.ts");
file('build/api.js', expand([ "build/genmeta.js", "lib" ]), { async: true }, function () {
    console.log("[P] generating build/api.js, localization.json and topiclist.json");
    runAndComplete([
        "node build/genmeta.js",
    ], this);
});

mkSimpleTask('build/addCssPrefixes.js', [ 'tools' ], "tools/addCssPrefixes.ts");
task('css-prefixes', [ "build/addCssPrefixes.js" ], { async: true }, function () {
    console.log("[P] modifying in-place all css files");
    runAndComplete([
        "node build/addCssPrefixes.js"
    ], this);
});

mkSimpleTask('build/shell.js', [ 'shell/shell.ts' ], "shell/shell.ts");
mkSimpleTask('build/package.js', [ 'shell/package.ts', 'build/shell.js' ], "shell/package.ts");
file('build/pkgshell.js', [ 'build/package.js' ], { async: true }, function () {
    console.log("[P] generating build/pkgshell.js and packaging");
    runAndComplete([
        "node build/package.js"
    ], this);
});

// Portal
mdkTask('build/portal/portal.html', ['portal/portal.mdk'], 'portal/portal.mdk');

// These dependencies have been hand-crafted by reading the various [refs.ts]
// files. The dependencies inside the same folder are coarse-grained: for
// instance, anytime something changes in [editor/], [build/editor.d.ts] gets
// rebuilt. This amounts to assuming that for all [foo/bar.ts], [bar.ts] is a
// dependency of [build/foo.js].
mkSimpleTask('build/browser.d.ts', [
    'browser/browser.ts'
], "browser/browser.ts");
mkSimpleTask('build/rt.d.ts', [
    'build/browser.d.ts',
    'rt',
    'lib'
], "rt/refs.ts");
mkSimpleTask('build/storage.d.ts', [
    'build/rt.d.ts',
    'rt/typings.d.ts',
    'build/browser.d.ts',
    'storage'
], "storage/refs.ts");
mkSimpleTask('build/ast.d.ts', [
    'build/rt.d.ts',
    'ast'
], "ast/refs.ts");
mkSimpleTask('build/libwab.d.ts', [
    'build/rt.d.ts',
    'rt/typings.d.ts',
    'build/browser.d.ts',
    'libwab'
], "libwab/refs.ts");
mkSimpleTask('build/libnode.d.ts', [
    'build/rt.d.ts',
    'rt/typings.d.ts',
    'libnode'
], "libnode/refs.ts");
mkSimpleTask('build/libcordova.d.ts', [
    'build/rt.d.ts',
    'rt/typings.d.ts',
    'build/browser.d.ts',
    'libcordova'
], "libcordova/refs.ts");
mkSimpleTask('build/editor.d.ts', [
    'rt/typings.d.ts',
    'build/browser.d.ts',
    'build/rt.d.ts',
    'build/ast.d.ts',
    'build/storage.d.ts',
    'build/libwab.d.ts',
    'build/libcordova.d.ts',
    'intellitrain',
    'editor'
], "editor/refs.ts");
mkSimpleTask('build/officemix.d.ts', [
    'officemix'
], "officemix/officemix.ts");
mkSimpleTask('build/jsonapi.d.ts', [], "noderunner/jsonapi.ts");
mkSimpleTask('build/client.js', [
    'rt/typings.d.ts',
    'build/jsonapi.d.ts',
    'tools/client.ts'
], "tools/client.ts");
// XXX coarse-grained dependencies here over the whole 'noderunner' directory
mkSimpleTask('build/nrunner.js', [
    'build/browser.d.ts',
    'rt/typings.d.ts',
    'build/rt.d.ts',
    'build/ast.d.ts',
    'build/libnode.d.ts',
    'noderunner'
], "noderunner/nrunner.ts");
// XXX same here
mkSimpleTask('build/runner.d.ts', [
    'build/browser.d.ts',
    'rt/typings.d.ts',
    'build/rt.d.ts',
    'build/storage.d.ts',
    'build/libwab.d.ts',
    'build/libnode.d.ts',
    'build/libcordova.d.ts',
    'runner'
], "runner/refs.ts");
mkSimpleTask('build/ace.js', [ "www/ace/ace-main.ts" ], "www/ace/refs.ts");

// Now come the rules for files that are obtained by concatenating multiple
// _js_ files into another one. The sequence exactly reproduces what happened
// previously, as there are ordering issues with initialization of global variables
// (unsurprisingly). Here's the semantics of these entries:
// - files ending with ".js" end up as dependencies (either as rule names, or as
//   statically-checked-in files in the repo, such as [langs.js]), and are
//   concatenated in the final file
// - files without an extension generate a dependency on the ".d.ts" rule and
//   the ".js" compiled file ends up in the concatenation
var concatMap = {
    "build/noderunner.js": [
        "build/browser",
        "build/rt",
        "build/ast",
        "build/api.js",
        "generated/langs.js",
        "build/libnode",
        "build/pkgshell.js",
        "build/nrunner.js",
    ],
    "build/runtime.js": [
        "build/rt",
        "build/storage",
        "build/libwab",
        "build/libnode",
        "build/libcordova",
        "build/runner",
    ],
    "build/main.js": [
        "build/rt",
        "build/ast",
        "build/api.js",
        "generated/langs.js",
        "build/storage",
        "build/libwab",
        "build/libcordova",
        "build/pkgshell.js",
        "build/editor" ,
    ],
};

// Just a dumb concatenation
function justCat(files, dest) {
    console.log("[C]", dest);
    var bufs = [];

    files.forEach(function (f) {
        bufs.push(fs.readFileSync(f));
    });

    fs.writeFileSync(dest, Buffer.concat(bufs));
}

// A concatenation that recomputes proper maps. Generates the source map in the
// same directory as the original map, and expects it to remain that way.
function mapCat(files, dest) {
  console.log("[C]", dest, "with maps");

  // An array of buffers for all the files we want to concatenate.
  var bufs = [];
  // Current line offest.
  var lineOffset = 0;
  // The source map we're generating.
  var map = new source_map.SourceMapGenerator({ file: dest + ".map" });

  files.forEach(function (f) {
    var buf = fs.readFileSync(f);
    bufs.push(buf);
    if (fs.existsSync(f + ".map")) {
      var originalMap = new source_map.SourceMapConsumer(fs.readFileSync(f + ".map", { encoding: "utf-8" }));
      originalMap.eachMapping(function (m) {
        map.addMapping({
          generated: {
            line: m.generatedLine + lineOffset,
            column: m.generatedColumn,
          },
          original: {
            line: m.originalLine,
            column: m.originalColumn,
          },
          source: m.source,
          name: m.name
        });
      });
      // An extra line was added for the sourcemap comment already.
      lineOffset--;
    }
    lineOffset += buf.toString().split("\n").length;
    if (buf[buf.length - 1] == "\n".charCodeAt(0))
        lineOffset--;
  });

  bufs.push(new Buffer("\n//# sourceMappingURL="+path.basename(dest)+".map"));

  fs.writeFileSync(dest, Buffer.concat(bufs));
  fs.writeFileSync(dest+".map", map.toString());
}

Object.keys(concatMap).forEach(function (f) {
    var isJs = function (s) { return s.substr(s.length - 3, 3) == ".js"; };
    var prelude = f == "build/noderunner.js" ? ["tools/node_prelude.js"] : [];
    var buildDeps = concatMap[f].map(function (x) { if (isJs(x)) return x; else return x + ".d.ts"; });
    var toConcat = prelude.concat(
      concatMap[f].map(function (x) { if (isJs(x)) return x; else return x + ".js"; })
    );
    file(f, buildDeps, { parallelLimit: branchingFactor }, function () {
      if (f == "build/main.js" && process.env.TD_SOURCE_MAPS)
        mapCat(toConcat, f);
      else
        justCat(toConcat, f);
    });
});

task('log', [], { async: false }, function () {
  if (process.env.TRAVIS_COMMIT_RANGE) {
    console.log("[I] dumping commit info to build/buildinfo.json");
    fs.writeFileSync('build/buildinfo.json', JSON.stringify({
      commit: process.env.TRAVIS_COMMIT,
      commitRange: process.env.TRAVIS_COMMIT_RANGE
    }));
  }
});

// Our targets are the concatenated files, which are the final result of the
// compilation. We also re-run the CSS prefixes thingy everytime.
desc('build the TypeScript projects')
task('default', [
  'css-prefixes',
  'build/client.js',
  'build/officemix.d.ts',
  'build/ace.js',
  'log'
].concat(Object.keys(concatMap)), {
  parallelLimit: branchingFactor,
},function () {
    console.log("[I] build completed.");
});

desc('clean up the build folders and files')
task('clean', [], function () {
    // XXX do this in a single call? check out https://github.com/mde/utilities/blob/master/lib/file.js
    generated.forEach(function (f) { jake.rmRf(f); });
    jake.rmRf('build/');
});

desc('run local test suite')
task('test', [ 'build/client.js', 'default', 'nw-build' ], { async: true }, function () {
  var task = this;
  console.log("[I] running tests")
  jake.exec([ 'node build/client.js buildtest' ],
    { printStdout: true, printStderr: true },
    function() { task.complete(); });
});

// this task runs as a "after_success" step in the travis-ci automation
desc('upload current build to the cloud')
task('upload', [], { async : true }, function() {
  var task = this;
  var upload = function (buildVersion) {
    var uploadKey = process.env.TD_UPLOAD_KEY || "direct";
    console.log("[I] uploading v" + buildVersion)
    jake.exec([ 'node build/client.js tdupload ' + uploadKey + ' ' + buildVersion ],
      { printStdout: true, printStderr: true },
      function() { task.complete(); });

  };
  if (!process.env.TRAVIS) {
    upload(process.env.TD_UPLOAD_USER || process.env.USERNAME);
  } else {
    assert(process.env.TRAVIS_BUILD_NUMBER, "missing travis build number");
    assert(process.env.TD_UPLOAD_KEY, "missing touchdevelop upload key");
    var buildVersion = 80100 + parseInt(process.env.TRAVIS_BUILD_NUMBER || - 80000);
    upload(buildVersion);
  }
})

desc('build portal html pages')
task('portal', ['build/portal/portal.html']);

desc('build and launch local server')
task('local', [ 'default' ], { async: true }, function() {
  var task = this;
  jake.mkdirP('build/local/node_modules')
  jake.cpR('build/shell.js', 'build/local/tdserver.js')
  process.chdir("build/local")
  var node = "node"
  if (process.env.TD_SHELL_DEBUG)
    node = "node-debug"
  jake.exec(
    [ node + ' tdserver.js --cli TD_ALLOW_EDITOR=true TD_LOCAL_EDITOR_PATH=../..' ],
    { printStdout: true, printStderr: true },
    function() { task.complete(); }
  );
});

task('nw', ['default', 'nw-npm'], { async: true }, function() {
  runAndComplete([ 'node node_modules/nw/bin/nw build/nw' ], this);
})

task('update-docs', [ 'build/client.js', 'default' ], { async: true }, function() {
  var task = this;
  jake.exec(
    [ 'node build/client.js updatehelp',
      'node build/client.js updatelang' ],
    { printStdout: true, printStderr: true },
    function() { task.complete(); }
  );
});

task('nw-npm', {async : true }, function() {
  var task = this;
  jake.mkdirP('build/nw');
  ['node-webkit/app.html',
   'node-webkit/logo.png',
   'node-webkit/client.ico',
   'node-webkit/package.json',
   'build/shell.js'].forEach(function(f) { jake.cpR(f, 'build/nw') })
  child_process.exec('npm install', { cwd: 'build/nw' },
      function (error, stdout, stderr) {
        if (stdout) console.log(stdout.toString());
        if (stderr) console.error(stderr.toString());
        if (error) task.fail(error);
        else task.complete();
      }
    );
})

task('nw-build', [ 'default', 'nw-npm' ], { async : true }, function() {
  var task = this;
  console.log('[I] building nw packages')
  var nwBuilder = require('node-webkit-builder');
  var nw = new nwBuilder({
      files: 'build/nw/**',
      platforms: ['win32', 'osx32', 'linux32'],
      buildDir: 'build/nw_build',
      cacheDir: 'nw_cache'
  });
  nw.on('log',  console.log);
  nw.build().then(function () {
     console.log('[I] nw packaged');
     task.complete();
  }).catch(function (error) {
    console.error(error);
    task.fail();
  });
});

task('cordova', [ 'default' ], {}, function () {
  // FIXME minify
  jake.mkdirP('build/cordova/');
  ['www/index.html',
   'build/browser.js',
   'build/main.js',
   'www/default.css',
   'www/editor.css'].forEach(function (f) { jake.cpR(f, 'build/cordova/'); });
});

// vim: ft=javascript
