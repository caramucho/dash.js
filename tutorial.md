#### dash开发笔记
开发环境
node -v
v12.13.1

dash.js
v3.0.0

grunt-cli
grunt-cli: The grunt command line interface (v1.3.2)

安装grunt
npm install
自动安装所有packages

### 开发调试
执行以下grunt命令
Build, watch file changes and launch samples page, which has links that point to reference player and to other examples (basic examples, captioning, ads, live, etc).
grunt dev

会有一个链接可以打开所有samples
Access URLs:
       Local: http://localhost:3000/samples/index.html
    External: http://136.187.81.50:3000/samples/index.html
          UI: http://localhost:3001
 UI External: http://136.187.81.50:3001

#### 添加custom的例子
samples文件夹创建custom-scenarios
修改samples.json文件
```json
{
    "section": "custom scenarios",
    "samples": [
        {
            "title": "added test scenario",
            "description": "test adding new abr test",
            "href": "custom-scenarios/abr/index.html"
        }
    ]
}
```
实时更新都可以反应在浏览器里

#### custom abr
参考advanced abr例子
无效化默认abr算法
添加新的abr rule文件LowestBitrateRule.js
```html
    <script class="code">
        function init() {
            var video,
                player,
                url = "https://dash.akamaized.net/envivio/EnvivioDash3/manifest.mpd";

            video = document.querySelector("video");
            player = dashjs.MediaPlayer().create();

            // don't use dash.js default rules
            player.updateSettings({
                'streaming': {
                    'abr': {
                        'useDefaultABRRules': false
                    }
                }
            });

            // add my custom quality switch rule. Look at LowestBitrateRule.js to know more
            // about the structure of a custom rule
            player.addABRCustomRule('qualitySwitchRules', 'LowestBitrateRule', LowestBitrateRule);

            player.initialize(video, url, true);
        }
    </script>
```

#### add abr rule
```js
var LowestBitrateRule;

// Rule that selects the lowest possible bitrate
function LowestBitrateRuleClass() {
    // ...
    function getMaxIndex(rulesContext) {
        // ...
    }
    instance = {
        getMaxIndex: getMaxIndex
    };

    setup();

    return instance;
}

LowestBitrateRuleClass.__dashjs_factory_name = 'LowestBitrateRule';
LowestBitrateRule = dashjs.FactoryMaker.getClassFactory(LowestBitrateRuleClass);
```
#### 获取现在的bitrate
```js
let StreamController = factory.getSingletonFactoryByName('StreamController');
let context = this.context;
let instance;
function getMaxIndex(rulesContext){
  // Get current bitrate
  let streamController = StreamController(context).getInstance();
  let abrController = rulesContext.getAbrController();
  let current = abrController.getQualityFor(mediaType, streamController.getActiveStreamInfo());

}
```
#### 获取各种metrics
```js
let MetricsModel = factory.getSingletonFactoryByName('MetricsModel');
function getMaxIndex(rulesContext){
  // here you can get some informations aboit metrics for example, to implement the rule
  let metricsModel = MetricsModel(context).getInstance();
  var mediaType = rulesContext.getMediaInfo().type;
  var metrics = metricsModel.getMetricsFor(mediaType, true);
}
```
#### metrics信息
BufferLevel: time, buffer level
BufferState:
DVBErrors:
DVRInfo:
DroppedFrames:
HttpList:
ManifestUpdate:
PlayList:
RepSwitchList:
RequestsQueue:
SchedulingInfo:
TcpList:
__proto__:

#### switch request
RuleClass的method要返回一个switch request
switchRequest.quality 目标bitrate
switchRequest.priority = SwitchRequest.PRIORITY.STRONG; 优先级
```js
let factory = dashjs.FactoryMaker;
let SwitchRequest = factory.getClassFactoryByName('SwitchRequest');

// If already in lowest bitrate, don't do anything
if (current === 0) {
    return SwitchRequest(context).create();
}

// Ask to switch to the lowest bitrate
let switchRequest = SwitchRequest(context).create();
switchRequest.quality = 0;
switchRequest.reason = 'Always switching to the lowest bitrate';
switchRequest.priority = SwitchRequest.PRIORITY.STRONG;
return switchRequest;
```

#### monitoring network metrics
buffer level, estimated bitrate
in init()
eventPoller setInterval 一定间隔执行event
```js
player.on(dashjs.MediaPlayer.events["PLAYBACK_ENDED"], function() {
    clearInterval(eventPoller);
    clearInterval(bitrateCalculator);
});

var eventPoller = setInterval(function() {
    var streamInfo = player.getActiveStream().getStreamInfo();
    var dashMetrics = player.getDashMetrics();
    var dashAdapter = player.getDashAdapter();

    if (dashMetrics && streamInfo) {
        const periodIdx = streamInfo.index;
        var repSwitch = dashMetrics.getCurrentRepresentationSwitch('video', true);
        var bufferLevel = dashMetrics.getCurrentBufferLevel('video', true);
        var bitrate = repSwitch ? Math.round(dashAdapter.getBandwidthForRepresentation(repSwitch.to, periodIdx) / 1000) : NaN;
        document.getElementById('bufferLevel').innerText = bufferLevel + " secs";
        document.getElementById('reportedBitrate').innerText = bitrate + " Kbps";
    }
}, 1000);

if (video.webkitVideoDecodedByteCount !== undefined) {
    var lastDecodedByteCount = 0;
    const bitrateInterval = 5;
    var bitrateCalculator = setInterval(function() {
        var calculatedBitrate = (((video.webkitVideoDecodedByteCount - lastDecodedByteCount) / 1000) * 8) / bitrateInterval;
        document.getElementById('calculatedBitrate').innerText = Math.round(calculatedBitrate) + " Kbps";
        lastDecodedByteCount = video.webkitVideoDecodedByteCount;
    }, bitrateInterval * 1000);
} else {
    document.getElementById('chrome-only').style.display = "none";
}
```
```html
<div>
        <strong>Reported bitrate:</strong>
        <span id="reportedBitrate"></span>
        <br />
        <strong>Buffer level:</strong>
        <span id="bufferLevel"></span>
        <div id="chrome-only">
            <strong>Calculated bitrate:</strong>
            <span id="calculatedBitrate"></span>
        </div>
</div>
```

