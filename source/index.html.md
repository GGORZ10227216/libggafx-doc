---
title: libggafx API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - shell
  - ruby
  - python
  - javascript

toc_footers:
  - <a href='#'>Sign Up for a Developer Key</a>
  - <a href='https://github.com/slatedocs/slate'>Documentation Powered by Slate</a>

includes:
  - errors

search: true

code_clipboard: true

meta:
  - name: description
    content: Documentation for the Kittn API
---

# 簡介

Libggafx是一個模擬Gameboy Advance(GBA) PPU的函式庫，此函式庫會依照系統當前的狀態繪製出相對應的畫面
# GBA graphics system specification

- GBA依靠一顆訂製的圖形核心Pixel Processing Unit(PPU)來處理以及繪製遊戲圖形，以下是其輸出規格:
	- 240 x 160 LCD 屏幕
	- 最大同時顯色數: 32768
	- 支持數種圖形特效:
		- [旋轉/縮放](#affine)
		- [半透明(α blending)](#alpha-blending)
		- [淡入淺出(fade-in/out)](#fade)
		- [馬賽克(mosiac)](#mosiac)
	- 96KBytes VRAM + 1KBytes Palette + 1KBytes OAM
	
<aside class="notice">
	有關memory layout，請參閱<a href="#vram-layout">VRAM layout</a>
</aside>

# PPU processing flowchart

- 擷取自 AGB Programming Manual Version 1.1 p15 CPU Block Diagram
<p align="center">
	<img src="https://lh3.googleusercontent.com/fife/AAbDypC7XHXLnbAgRNjNl9T0HITXat5RNv7VoNs8C7kwUla4ffKY5Rw6YIRgN5Rc6HPFV8BqO9QjbY4EswrCwDjF7JUACxEGWsbs17ya5kwj7T86MsNU8l5U6_JHYRSNGFEdl1Xj--bplVy6UmxOV0aGYOVYaKzCPKhffUA2CDPnubDmMPOavWuyKSt9y5l1Z7IcrcVC7ryhU0_xkwcyU0Y8yDMrFY6YgmON9ja0bYwM50ntJAzBbI0vkOr-ITkd0CU_SuRgwRpNF-dIMz4ryOmZ99BB7KLqx9tHDAEC6MQuPLRY7RnC2mhZsV9JFzGCudBNC2egCBxranKz8pEHR_7Dd7pcEbmktrhP4v6AD0JD9o7Hidhm5eFDYqWCyGoxmpFz05QaEc0B7U96Q1FR8zNbrK2Z4e7H5NN1RoWYKzc3uTFJIDupwQC0cytVvM3Q1Z6C6QIUPT88cmREWywvWtXXsD2XW5M12jpLxTeJVLhJXOlyJUWHOJeXZjPcu87I7cdX5V_S_vTWipE4P5-lx_BLKgmsOhaJmvThu--ubeuor2mC2TsrU-K2NJb89FaSKccnMkxsEyM_sKSuw_FdTJz2_eYq9EjRFQp8nSldhMdDVHuHwXmHOVAIJ65KrjldMOguKRJxSXNDn1HZ29VCRnaw_QRs9ZfY3hm8FAD7UUzRntXoJxGqzmjcA_o2XYZMx1Ov8le6Q4FdD1xtGDiv1DJ_rf3D-LeGXvq6gltdw-JIqdQxB-4baScdz41QRNR5GskeYbc_FzyXy_AhnsxVBAbkSf2k5YDQL3ESR3-nCtYYzmpHTeLvOnhLE1slRh5lqL_UkXCiC7oMq230tYa-JZRr2TPKzDMATk3GU6srI7hNB1CrHe9F7pCURCXiT76bwsvEqKf-9Ydylo8FzvDw15LpTKLKx6OyOKDY1XxOhDQvrkPSFYN9eR2G2KjGW5poXcQTfEmznEudj4rrBJEuR17qSjzB11TRg_iygh_otA5OIxQANRXnB5L0iUC7-oAdZ24hch9_Ulmp2wzHB3aGNyQ_pgFN61dDfzn5m8BYMY25-oTotFZiy3l4FATOE8pp0J-8RdjngnRkjgX8TMxcvIu2W7VB86e8XjBX2wQSOdGugIPdtLGext73iuaEUJAbT4kQ01XWz2FKcTow-pJqpwLHWCshGBzj6JdJdFboOcqYdyskd7efyBdwfQKx0C3ERM6fZbna49XRQuMadU9EcqWAMLLdeKA2BiPH2keDXyWtJl4x5WqPOCGqviB_Ujs2_a4MhFa38rp1kcrLVDCPL13OZSZmvpAWYmn4E6Sq91sYQR4MOcuJgziC4L3Jv2gHKkW2tFl7aw-kJfIM9AJpMR9OS-ATdVfyLtIkVKWGVsOxbiaBNzxfOHxC7VVIDvNuDtzlCyvE9ZE9U6wrXwsToqEytEY0tznseugepUpIj8ud0rEl4YpUIXO9vGWbOkdMb4az0bW4chNFefalbNanQ94=w958-h911">
</p>
- 不難看出每一幀遊戲畫面都是依照以下的處理流程在進行繪製:
	1. 繪製背景圖層
	2. 繪製Sprite(OBJ)
	3. 進行z index checking
		- 就是依照前後順序來疊合圖層
	4. 如果是Mode[0, 1, 2]則使用palette上色，反之為bitmap mode，跳過上色
	5. 特殊效果處理

# Background drawing

GBA在繪製Background(BG)圖層，一共有6種mode，mode [0, 1 ,2]為基於character的tile mode
mode [4 ,5, 6]則是直接繪製pixel的bitmap mode，以下會個別解釋
## Character Mode (BG Mode 0-2)

此模式會利用預先準備好的Character來組合出各BG圖層的內容，此模式的工作效率較bitmap模式來得高，因此大部分的商業遊戲都會使用此模式進行遊戲畫面繪製

<aside class="notice">
character我們可以把他理解成一個大小為8*8的小圖塊，遊戲程式會在需要的時候將這些character搬移進VRAM中進行繪製，詳細請參閱<a href="#Character-drawing">Character drawing</a>
</aside>

> To authorize, use this code:

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
```

```shell
# With shell, you can just pass the correct header with each request
curl "api_endpoint_here" \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
```

> Make sure to replace `meowmeowmeow` with your API key.

Kittn uses API keys to allow access to the API. You can register a new Kittn API key at our [developer portal](http://example.com/developers).

Kittn expects for the API key to be included in all API requests to the server in a header that looks like the following:

`Authorization: meowmeowmeow`

<aside class="notice">
You must replace <code>meowmeowmeow</code> with your personal API key.
</aside>

# Kittens

## Get All Kittens

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get()
```

```shell
curl "http://example.com/api/kittens" \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let kittens = api.kittens.get();
```

> The above command returns JSON structured like this:

```json
[
  {
    "id": 1,
    "name": "Fluffums",
    "breed": "calico",
    "fluffiness": 6,
    "cuteness": 7
  },
  {
    "id": 2,
    "name": "Max",
    "breed": "unknown",
    "fluffiness": 5,
    "cuteness": 10
  }
]
```

This endpoint retrieves all kittens.

### HTTP Request

`GET http://example.com/api/kittens`

### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
include_cats | false | If set to true, the result will also include cats.
available | true | If set to false, the result will include kittens that have already been adopted.

<aside class="success">
Remember — a happy kitten is an authenticated kitten!
</aside>

## Get a Specific Kitten

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get(2)
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get(2)
```

```shell
curl "http://example.com/api/kittens/2" \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let max = api.kittens.get(2);
```

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "name": "Max",
  "breed": "unknown",
  "fluffiness": 5,
  "cuteness": 10
}
```

This endpoint retrieves a specific kitten.

<aside class="warning">Inside HTML code blocks like this one, you can't use Markdown, so use <code>&lt;code&gt;</code> blocks to denote code.</aside>

### HTTP Request

`GET http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description 
--------- | ----------- 
ID | The ID of the kitten to retrieve

## Delete a Specific Kitten

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.delete(2)
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.delete(2)
```

```shell
curl "http://example.com/api/kittens/2" \
  -X DELETE \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let max = api.kittens.delete(2);
```

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "deleted" : ":("
}
```

This endpoint deletes a specific kitten.

### HTTP Request

`DELETE http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the kitten to delete

