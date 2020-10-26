# arcacon-downloader
arcacon-downloader

```dart
import 'dart:io';

import 'package:arcacon/download/download_task.dart';
import 'package:arcacon/download/native_downloader.dart';
import 'package:arcacon/other/dialogs.dart';
import 'package:arcacon/other/html/parser.dart';
import 'package:cached_network_image/cached_network_image.dart';
import 'package:ext_storage/ext_storage.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';
import 'package:flutter/services.dart';
import 'package:flutter_ffmpeg/flutter_ffmpeg.dart';
import 'package:path_provider/path_provider.dart';
import 'package:permission_handler/permission_handler.dart';
import 'package:http/http.dart' as http;
import 'package:video_player/video_player.dart';
import 'package:path/path.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  var appdir = await getApplicationDocumentsDirectory();
  if (Platform.isAndroid) {
    if (!appdir.path.contains('/com.violet.arcacondownloader/')) return;
  }

  runApp(
    MaterialApp(
      title: '아카콘 다운로더',
      home: MainPage(),
      theme:
          ThemeData(brightness: Brightness.dark, primarySwatch: Colors.orange),
    ),
  );
}

class MainPage extends StatefulWidget {
  @override
  _MainPageState createState() => _MainPageState();
}

class _MainPageState extends State<MainPage> {
  TextEditingController textEditingController = TextEditingController();
  int _value = 0;
  int _sort = 0;
  String curUrl = 'https://arca.live/e/?';
  int _page = 0;
  List<Arcacon> _lists = List<Arcacon>();
  Map<String, VideoPlayerController> _videos =
      Map<String, VideoPlayerController>();
  ScrollController _controller = ScrollController();
  bool scrollOnce = false;

  @override
  void initState() {
    // TODO: implement initState
    super.initState();

    WidgetsBinding.instance.addPostFrameCallback((_) => _load());

    _controller.addListener(() {
      if (scrollOnce) return;
      if (_controller.offset > _controller.position.maxScrollExtent / 4 * 3) {
        scrollOnce = true;
        Future.delayed(Duration(milliseconds: 100)).then((value) async {
          _page++;
          await _load();
          scrollOnce = false;
        });
      }
    });
  }

  @override
  void didChangeDependencies() async {
    super.didChangeDependencies();

    if (await Permission.storage.isPermanentlyDenied ||
        await Permission.storage.isUndetermined ||
        await Permission.storage.isDenied) {
      if (await Permission.storage.request() == PermissionStatus.denied) {
        await Dialogs.okDialog(
            this.context, "저장공간 권한을 허용하지 않으면 앱을 이용할 수 없습니다.");
        if (Platform.isAndroid)
          SystemNavigator.pop();
        else if (Platform.isIOS) exit(0);
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('아카콘 다운로더'),
        actions: [
          IconButton(
            padding: EdgeInsets.zero,
            icon: Text(
              _sort == 0 ? '등록순' : '판매순',
              style: TextStyle(fontSize: 16.0),
            ),
            onPressed: () async {
              setState(() {
                _sort = 1 - _sort;
                _page = 0;
                _lists.clear();
              });
              await _load();
            },
          ),
          IconButton(
            icon: Icon(Icons.info_outline),
            onPressed: () async {
              await showDialog(
                context: context,
                builder: (BuildContext context) {
                  return InfoPage();
                },
              );
            },
          ),
        ],
      ),
      body: CustomScrollView(
        physics: const BouncingScrollPhysics(),
        controller: _controller,
        slivers: <Widget>[
          SliverPersistentHeader(
            floating: true,
            delegate: AnimatedOpacitySliver(
              minExtent: 64 + 12.0,
              maxExtent: 64.0 + 12,
              searchBar: Padding(
                padding: EdgeInsets.symmetric(vertical: 8.0, horizontal: 12.0),
                child: Card(
                  child: Padding(
                    padding: EdgeInsets.symmetric(horizontal: 16.0),
                    child: Row(
                      children: [
                        Icon(Icons.search),
                        Container(width: 8.0),
                        DropdownButton(
                            value: _value,
                            items: [
                              DropdownMenuItem(
                                child: Text("제목",
                                    style: TextStyle(fontSize: 20.0)),
                                value: 0,
                              ),
                              DropdownMenuItem(
                                child: Text("판매자",
                                    style: TextStyle(fontSize: 20.0)),
                                value: 1,
                              ),
                            ],
                            onChanged: (value) {
                              setState(() {
                                _value = value;
                              });
                            }),
                        Container(width: 8.0),
                        Expanded(
                          child: TextField(
                            controller: textEditingController,
                            textInputAction: TextInputAction.go,
                            decoration: new InputDecoration(
                              hintText: "검색어를 입력하세요",
                              contentPadding: const EdgeInsets.all(0.0),
                              focusedBorder: InputBorder.none,
                              enabledBorder: InputBorder.none,
                              errorBorder: InputBorder.none,
                              disabledBorder: InputBorder.none,
                            ),
                            style: TextStyle(fontSize: 20.0),
                            onEditingComplete: () async {
                              FocusScope.of(context).unfocus();

                              var url = 'https://arca.live/e/?keyword=' +
                                  Uri.encodeFull(textEditingController.text);

                              if (_value == 0)
                                url += '&target=title';
                              else if (_value == 1) url += '&target=nickname';

                              curUrl = url;

                              _page = 0;
                              _lists.clear();

                              await _load();
                            },
                          ),
                        ),
                        IconButton(
                          icon: Icon(Icons.arrow_back),
                          onPressed: () {
                            textEditingController.text = '';
                          },
                        ),
                      ],
                    ),
                  ),
                ),
              ),
            ),
          ),
          SliverPadding(
            padding: EdgeInsets.fromLTRB(12, 0, 12, 16),
            sliver: SliverGrid(
              gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
                crossAxisCount: 2,
                crossAxisSpacing: 8,
                mainAxisSpacing: 8,
                childAspectRatio: 1,
              ),
              delegate: SliverChildBuilderDelegate(
                (BuildContext context, int index) {
                  var e = _lists[index];

                  if (e.image.endsWith('.mp4')) {
                    if (!_videos.containsKey(e.image)) {
                      var vc = VideoPlayerController.network(e.image);
                      _videos[e.image] = vc;
                      vc.initialize().then((value) async {
                        await _videos[e.image].setLooping(true);
                        Future.delayed(Duration(milliseconds: 1000)).then(
                            (value) async => await _videos[e.image].play());
                        setState(() {});
                      });
                    } else {
                      Future.delayed(Duration(milliseconds: 500))
                          .then((value) async => await _videos[e.image].play());
                    }
                  }

                  return Card(
                    child: InkWell(
                      child: Column(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: [
                          !e.image.endsWith('.mp4')
                              ? CachedNetworkImage(imageUrl: e.image)
                              : _videos[e.image].value.initialized
                                  ? Container(
                                      height: 100.0,
                                      child: AspectRatio(
                                        aspectRatio:
                                            _videos[e.image].value.aspectRatio,
                                        child: VideoPlayer(
                                          _videos[e.image],
                                        ),
                                      ))
                                  : Container(),
                          Text(
                            e.title,
                            textAlign: TextAlign.center,
                            style: TextStyle(fontWeight: FontWeight.bold),
                          ),
                          Padding(
                            padding: EdgeInsets.symmetric(horizontal: 4.0),
                            child: Text(
                              e.uploader,
                              softWrap: true,
                              maxLines: 1,
                              overflow: TextOverflow.ellipsis,
                              textAlign: TextAlign.center,
                            ),
                          ),
                          Text(
                            e.count,
                            textAlign: TextAlign.center,
                          ),
                        ],
                      ),
                      onTap: () async {
                        // Navigator.push(
                        //   context,
                        //   MaterialPageRoute(
                        //       builder: (context) => ArcaconPage(
                        //             arcacon: e,
                        //           )),
                        // );
                        // showDialog(context: null)

                        var dir = (await getTemporaryDirectory());

                        var files = await showDialog(
                          context: context,
                          builder: (BuildContext context) {
                            return DownloadPage(
                              arcacon: e,
                              basePath: dir.path,
                            );
                          },
                        );

                        Navigator.push(
                          context,
                          MaterialPageRoute(
                              builder: (context) => ArcaconPage(
                                    arcacon: e,
                                    files: files,
                                  )),
                        );
                      },
                    ),
                  );
                },
                childCount: _lists.length,
              ),
            ),
          ),
        ],
      ),
    );
  }

  _load() async {
    var url = curUrl;
    if (_sort == 1) url = 'https://arca.live/e/?sort=rank';
    url += '&p=' + (_page + 1).toString();

    var html = (await http.get(url)).body;
    _lists.addAll(await ArcaliveParser.parseArcaconList(html));

    setState(() {});
  }
}

class DownloadPage extends StatefulWidget {
  final Arcacon arcacon;
  final String basePath;

  DownloadPage({this.arcacon, this.basePath});

  @override
  _DownloadPageState createState() => _DownloadPageState();
}

class _DownloadPageState extends State<DownloadPage> {
  int downloadedFileCount = 0;
  bool isGif = false;
  int gifCount = 0;
  int gifComplete = 0;
  List<String> _contents = List<String>();

  @override
  void initState() {
    // TODO: implement initState
    super.initState();
    Future.delayed(Duration(milliseconds: 100)).then((value) async {
      var html = (await http.get(widget.arcacon.url)).body;
      _contents = await ArcaliveParser.parseArcacon(html);

      var path = widget.basePath;

      if (!Directory(path).existsSync()) await Directory(path).create();

      var downloader = await NativeDownloader.getInstance();

      var tasks = List<DownloadTask>.generate(
          _contents.length,
          (i) => DownloadTask(
              url: _contents[i],
              filename: i.toString().padLeft(2, '0') +
                  '.' +
                  _contents[i].split('.').last));

      await downloader.addTasks(tasks.map((e) {
        e.downloadPath = join(
            join(path, widget.arcacon.title.replaceAll('/', ' '), e.filename));

        e.startCallback = () {};
        e.completeCallback = () {
          setState(() {
            downloadedFileCount++;
          });
        };

        return e;
      }).toList());

      while (tasks.length != downloadedFileCount) {
        await Future.delayed(Duration(milliseconds: 500));
      }

      for (int i = 0; i < tasks.length; i++) {
        var element = tasks[i];
        if (element.url.endsWith('.mp4')) {
          gifCount++;
          isGif = true;
        }
      }

      setState(() {});

      for (int i = 0; i < tasks.length; i++) {
        var element = tasks[i];
        var fn = join(join(
            path, widget.arcacon.title.replaceAll('/', ' '), element.filename));
        if (element.url.endsWith('.mp4')) {
          await FlutterFFmpeg().executeWithArguments([
            '-y',
            '-i',
            fn,
            '-loop',
            '0',
            fn.replaceAll('.mp4', '.gif'),
          ]);
          setState(() {
            gifComplete++;
          });
        }
      }

      for (int i = 0; i < tasks.length; i++) {
        var element = tasks[i];
        var fn = join(join(
            path, widget.arcacon.title.replaceAll('/', ' '), element.filename));
        if (element.url.endsWith('.mp4')) {
          await File(fn).delete();
        }
      }

      Navigator.pop(
          this.context,
          tasks
              .map((e) => join(join(path,
                      widget.arcacon.title.replaceAll('/', ' '), e.filename))
                  .replaceAll('.mp4', '.gif'))
              .toList());
    });
  }

  @override
  Widget build(BuildContext context) {
    return WillPopScope(
      onWillPop: () async {
        return false;
      },
      child: Container(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          crossAxisAlignment: CrossAxisAlignment.center,
          children: <Widget>[
            Card(
              color: Color(0xFF353535),
              child: SizedBox(
                child: Container(
                  padding: EdgeInsets.fromLTRB(20, 40, 20, 20),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.center,
                    children: <Widget>[
                      SizedBox(
                        child: CircularProgressIndicator(),
                        width: 50,
                        height: 50,
                      ),
                      Container(
                        height: 16,
                      ),
                      Text(isGif
                          ? '변환중...[$gifComplete/$gifCount]'
                          : '다운로드 중...[$downloadedFileCount/${_contents.length}]'),
                    ],
                  ),
                  width: 180,
                  // height: 190,
                ),
              ),
            ),
          ],
        ),
        decoration: BoxDecoration(
          borderRadius: BorderRadius.all(Radius.circular(1)),
          boxShadow: [
            BoxShadow(
              color: Colors.black.withOpacity(0.4),
              spreadRadius: 1,
              blurRadius: 1,
              offset: Offset(0, 3), // changes position of shadow
            ),
          ],
        ),
      ),
    );
  }
}

class ArcaconPage extends StatefulWidget {
  final Arcacon arcacon;
  final List<String> files;

  ArcaconPage({this.arcacon, this.files});

  @override
  _ArcaconPageState createState() => _ArcaconPageState();
}

class _ArcaconPageState extends State<ArcaconPage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.arcacon.title),
        actions: [
          IconButton(
            icon: Icon(Icons.file_download),
            onPressed: () async {
              var dir = join(
                  await ExtStorage.getExternalStoragePublicDirectory(
                      ExtStorage.DIRECTORY_DOWNLOADS),
                  '아카콘');
              await showDialog(
                context: context,
                builder: (BuildContext context) {
                  return DownloadPage(
                    arcacon: widget.arcacon,
                    basePath: dir,
                  );
                },
              );

              Dialogs.okDialog(context, '다운로드 완료!');
            },
          )
        ],
      ),
      body: CustomScrollView(
        physics: BouncingScrollPhysics(),
        // primary: false,
        slivers: <Widget>[
          SliverGrid(
            gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
              crossAxisCount: 4,
              // crossAxisSpacing: 8,
              // mainAxisSpacing: 8,
              childAspectRatio: 1,
            ),
            delegate: SliverChildBuilderDelegate(
              (BuildContext context, int index) {
                return Image.file(File(widget.files[index]));
              },
              childCount: widget.files.length,
            ),
          ),
        ],
      ),
    );
  }
}

class InfoPage extends StatelessWidget {
  Color getColor(int i) {
    return Colors.grey.shade400;
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () {
        Navigator.pop(context);
      },
      child: Container(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          crossAxisAlignment: CrossAxisAlignment.center,
          children: <Widget>[
            Card(
              color: Color(0xFF353535),
              child: SizedBox(
                child: Container(
                  padding: EdgeInsets.fromLTRB(20, 0, 20, 0),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: <Widget>[
                      Text(''),
                      Text('아카콘 다운로더',
                          style: TextStyle(
                              fontSize: 30, fontWeight: FontWeight.bold)),
                      Text('2020.10.26', style: TextStyle(fontSize: 20)),
                      Text('by Koromo', style: TextStyle(fontSize: 20)),
                      Text(''),
                      // Text('Project-Violet Android App'),
                      // Text(
                      //   Translations.of(context).trans('infomessage'),
                      //   textAlign: TextAlign.center,
                      //   style: TextStyle(fontSize: 10),
                      // ),
                    ],
                  ),
                  width: 250,
                  // height: 190,
                ),
              ),
            ),
          ],
        ),
        decoration: BoxDecoration(
          borderRadius: BorderRadius.all(Radius.circular(1)),
          boxShadow: [
            BoxShadow(
              color: Colors.black.withOpacity(0.4),
              spreadRadius: 1,
              blurRadius: 1,
              offset: Offset(0, 3), // changes position of shadow
            ),
          ],
        ),
      ),
    );
  }
}

class AnimatedOpacitySliver implements SliverPersistentHeaderDelegate {
  AnimatedOpacitySliver(
      {this.minExtent, @required this.maxExtent, this.searchBar});
  final double minExtent;
  final double maxExtent;

  Widget searchBar;

  @override
  Widget build(
      BuildContext context, double shrinkOffset, bool overlapsContent) {
    return searchBar;
    // return Stack(
    //   fit: StackFit.expand,
    //   children: [
    //     AnimatedOpacity(
    //       child: searchBar,
    //       opacity: 1.0 - max(0.0, shrinkOffset - 20) / (maxExtent - 20),
    //       duration: Duration(milliseconds: 100),
    //     )
    //   ],
    // );
  }

  @override
  bool shouldRebuild(SliverPersistentHeaderDelegate oldDelegate) {
    return true;
  }

  @override
  FloatingHeaderSnapConfiguration get snapConfiguration => null;

  @override
  OverScrollHeaderStretchConfiguration get stretchConfiguration => null;
}

class Arcacon {
  String image;
  String url;
  String title;
  String count;
  String uploader;

  Arcacon({this.image, this.uploader, this.url, this.count, this.title});
}

class ArcaliveParser {
  static Future<List<Arcacon>> parseArcaconList(String html) async {
    var doc = (await compute(parse, html))
        .querySelector(
            'body > div > div.content-wrapper.clearfix > article > div > div > div.emoticon-list')
        .querySelectorAll('a')
        .where((element) => element.attributes['href'].contains('p='))
        .toList();

    return doc
        .map((e) => Arcacon(
              url: 'https://arca.live' + e.attributes['href'],
              title: e.querySelector('div.title').text,
              count: e.querySelector('div.count').text,
              uploader: e.querySelector('div.maker').text,
              image: 'https:' +
                  (e.querySelector('img') == null
                      ? e.querySelector('video').attributes['src']
                      : e.querySelector('img').attributes['src']),
            ))
        .toList();
  }

  static Future<List<String>> parseArcacon(String html) async {
    return (await compute(parse, html))
        .querySelector(
            'body > div > div.content-wrapper.clearfix > article > div > div.article-wrapper > div.article-body')
        .children
        .where((element) => element.attributes.containsKey('src'))
        .map((element) => 'https:' + element.attributes['src'])
        .toList();
  }
}
```
