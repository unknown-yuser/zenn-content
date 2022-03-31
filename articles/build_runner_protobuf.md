---
title: "Build_RunnerでProtobufのコンパイル/コード生成"
emoji: ""
type: "tech"
topics: ["Flutter", "Dart", "protobuf", "build_runner"]
published: false
---

# はじめに

protoファイルに対して `protoc --dart_out=grpc:path/to/generated -I protos source.proto` というコマンドを実行すると dartファイルが自動生成される。しかし、dart や flutter には **build_runner** というdartコードを自動生成するパッケージがあり、これを使うことがdartの文化に沿うように思える。そこで proto ファイルを build_runner を通してdartコードを生成する方法を調査した。

# 準備

`protoc` は呼び出せるようにする。

# プロジェクト構成

dartコードを自動生成するためには `build.yaml` が必要。`build.yaml` ではdartコードを自動生成するための関数を指定したり、コンパイルの対象となるディレクトリを指定することができる。

```
yourApp
├─android
.
.
.
├─lib
│  ├─builder.dart
│  └─main.dart
├─proto
│  └─hello.proto
├─test
├─build.yaml
├─pubspec.yaml
.
.
.
```

# build.yaml

<!-- TODO: 以下の説明を追加
    1. myBuilderGenerator とは?
    2. buiderの取得方法
    3. 拡張子によるフィルター
    4. auto_apply / build_to の説明
    5. 出力先を指定するために options を追加
    6. ソースを指定するために targets を追加
 -->

```.yaml
builders:
  # Generator for protobuf
  mybuilderGenerator:
    import: 'package:yourApp/builder.dart'
    builder_factories: ['mybuilderGenerator']
    build_extensions: {
      '.proto': ['.pbjson.dart', '.pb.dart', '.pbenum.dart']
    }
    auto_apply: root_package
    build_to: source
    defaults:
      options: {'output': 'lib' }
targets:
  $default:
    sources:
      - lib/**
      - proto/**
      - $package$

```

# builder.dart

<!-- TODO:
    1. builder の説明
    2. scratch の説明
    3. buildExtensions の実装の説明 (重複するがよくわかっていない)
    4. protocでコンパイルしてコピーして消していることを説明
        - Optionから消す先を指定している
 -->

```
import 'dart:async';
import 'dart:io';

import 'package:build/build.dart';
import 'package:glob/glob.dart';
import 'package:scratch_space/scratch_space.dart';
import 'package:path/path.dart' as p;

Builder mybuilderGenerator(BuilderOptions options) =>
    MyBuilder(options);

class FrecoMessageBuilder implements Builder {
  final BuilderOptions options;
  final List<String> deleteFileList = <String>[];

  MyBuilder(this.options);

  late final _scratch = Resource(() => ScratchSpace(),
      dispose: (o) => o.delete(),
      beforeExit: () => {
            for (final file in deleteFileList) {File(file).delete()}
          });
  final _extensions = <String>[
    '.pbjson.dart',
    '.pb.dart',
    '.pbenum.dart',
  ];

  @override
  Map<String, List<String>> get buildExtensions => {'.proto': _extensions};

  @override
  FutureOr<void> build(BuildStep buildStep) async {
    final space = await buildStep.fetchResource(_scratch);
    final protoAssets = await buildStep
        .findAssets(Glob('**/*.proto'))
        .fold<List<AssetId>>(
            <AssetId>[], (previous, element) => previous..add(element));
    await space.ensureAssets(protoAssets, buildStep);

    final file = space.fileFor(buildStep.inputId);
    if (!file.existsSync()) {
      print("${file.path} は存在しませんでした。");
      return;
    }

    final dirpath = Directory.current.path;
    final protoDir = p.relative(p.dirname(file.path), from: dirpath);

    final process = await Process.run(
        'protoc',
        [
          '--dart_out=grpc:$protoDir',
          '-I$protoDir',
          p.relative(file.path, from: dirpath)
        ],
        workingDirectory: dirpath);

    final code = process.exitCode;
    if (code != 0) {
      print(process.stderr);
      return;
    }

    for (final ext in _extensions) {
      final extAssetId = buildStep.inputId.changeExtension(ext);
      final extOutputFile = space.fileFor(extAssetId);

      if (await extOutputFile.exists()) {
        await space.copyOutput(extAssetId, buildStep);
        final srcPath = File(extAssetId.path);
        final destPathSegments = <String>[
          ...options.config["output"].split("/"),
          p.basename(extAssetId.path)
        ];
        await srcPath.copy(p.joinAll(destPathSegments));
        deleteFileList.add(extAssetId.path);
      } else {
        print('No output for: ${extAssetId.path}: $extOutputFile.path} found');
      }
    }
  }
}
```

# 参考

- https://zenn.dev/umatoma/articles/73a24ffe46b33e
- https://github.com/dart-lang/build/blob/master/docs/build_yaml_format.md
- https://github.com/google/protobuf.dart/issues/158