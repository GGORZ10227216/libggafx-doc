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
- 更準確地來說，每一幀顯示畫面都是逐行繪製的
	- 有一些特殊效果必須依賴這個特性才能正確顯示，所以想要投機取巧的一次畫完整個畫面應該是**不可行**
	- 每一行(Row)繪製完畢後會觸發系統的H-Blank，每一幀繪製完畢後會觸發系統的V-Blank，詳情請參閱[PPU interrupt mechanism](#PPU-interrupt-mechanism)
- libggafx的做法是用一個drawing cursor，一路從[0, 0]繪製到[239, 0]，接著再換到[0, 1]~[239, 1]，最終整個幀的繪製會結束於[239, 159]
	- 對於每一row，libggafx都會分別繪製出所有的BG row & Sprite row, 再根據各圖層的z index做疊合，最後進行特效處理
# Background drawing

GBA在繪製Background(BG)圖層，一共有6種mode，mode [0, 1 ,2]為基於character的tile mode
mode [4 ,5, 6]則是直接繪製pixel的bitmap mode，以下會個別解釋
## Character Mode (BG Mode 0-2)

Character Mode是統稱，泛指Mode 0, 1, 2這三種基於character顯示的繪製模式，這三種模式在使用上也有點差異，詳見下表

<p align="center">
	<img src="https://lh3.googleusercontent.com/fife/AAbDypAFWtrthCQIW-q-AecbUAusMM85aCUOOHOnuvu71ZYulHxafE7PNcPu3swv5Tsh5JXgT115icBo9l7_XhA3SdHNTnb_e2clt7A-Ehxud4DFbPRGHB49A1Rhf7SwgS54c4-AB2qWPfLKJvpAxBA8vsGVg3jXQC4tLEg0KeqHsBhpOUYdy7PRY_55GxzeW5mG6PCgT3KUnSQT_lm1H08POmz-HTfZsZv3nHjdOmU5QXgHe3gQP8XuozXvvyOnVUybkDavP2M78-4xTlVoC55hm8odRvvzWXdT_PJT9gzcyWi-8jBOzOFgu0_3w64mFjqOd81kW5oyN-vaz9JxZbi1Y7SvK6rYhXIEJAEizNhS92ZIWScgn1EO5yqYDvogz62BO2BleRLQmJSTaBETkcOq6B-4HIntr8xTZ86-TlSWW_RB_vg39SiBqnGxZu3A-0ELF-x_iLfDLx8CXYOVKthjnt4MLlMKvx907GFj2jf0InkxkxhJ82aOM1X1JkKO2ZnSfRuF7_-LGYJHbPmpsxKC8h_T5up_ABCurAv5SW8fSW2QkeM9BZHsxJkUWZiUk1XtLZfaaLTVZi1d0dsb7TLfn6HjqP3FhGTuWjQDO4zjEWn_lGeLUuuTOQlB2oXYc2eVYUmJkhha3-ItYxOk6aeiEsH3YYhXYoLAWdB8lEU8J5fV-0uLOYQlbzKgySl6LzwJoadCkmMbu_g9Qh5HXCRDCY0rqu9hsaZaBaED_C7B1iew1Wjr7stfhwzeKo31LOdq9ON8ZVGH74ikw0h1F2tcUw8G1rCJ-K1R92ny2KOKtbFj-nnSPOpR3BP3AL0dV32Gp1LOoMp0d6-gzHtnDAdMDJVNk4o_OzoRUrjdL1G8gPYsd0XI6dR4_534l7E70sf7TCTmVAaYTtx-L9dS4ABBhUZWZKfBC64mMEa6dh75VSgk3q-X2S9JfSr0Hgzd9pUBpDSs2my5Yzjy30GXDU76LhCz2Z0fZvbw2UxS9s7H-1Ir0_jrcm2_VLA57AAuj1vyBaAT8uTswBpYkblmkbUcDorPiojHbA6ACZY-OmMgXLQrU9Z1r_IGsCqRSq6UlgFT3Hltv-k5p0TXlJ8wttPcVL5s74hFVBPMUWrjCvr2ibzjdKW1JT6qsWffNkqLIfeMemGIJUKfzDyT3TjvlYzudRlr8ntGjEqh2Zgqm-XcD0-xfN6NHrBC7MnVMZY85SqfxMnAbNAGYK9BYhojrxaGsAWKh4sGRXIKvgvmC7lG0pylFObXMpPnEUSCkKluQ4DZ7MmTT_Y3oXPTMtC2DQqFS62w5SZHdMySTKDTUpW40MqjkQf1dYuFMrIH_75ONoZrlkebLFbuSJQQ4YtJBkZ0paD-VJ6oAoB7F_ppEO0xhdlZt8_-zb4hCyRPCu8QMr7_LIMoGZY1lOV00c2JkD8WYt4rETgzZLD6DiwDMrSbOJcPZiC7ZVr-UmziRCXf0D79c6awtEdXLEtkAbzbdL8=w964-h958">
</p>
<aside class="notice">
Feature:
	<ol>
		<li>[BG圖層]水平/垂直捲動</li>
		<li>[character]水平/垂直翻轉</li>
		<li>[BG/Character]馬賽克效果</li>
		<li>[BG/Character]alpha blending</li>
		<li>[BG/Character]淡入淺出</li>
		<li>[BG/Character]z index</li>
	</ol>
</aside>

### Drawing step of character mode background

- PPU在繪製每一個character mode background圖層時，都必須要參考:
	- [Screen memory](#Screen-memory)
		- 用來描述Character如何在圖層中分布
	- [Character memory](#Character-memory)
		- 描述組成圖形的基本8*8圖塊(tile)之內容
- libggafx會依照以下的步驟進行每一層的Background圖層繪製
	1. 參考該圖層的[BGCNT](#BGCNT)來解析Screen memory
	2. 在實際繪製的過程中我們會有兩個cursor
		- Drawing cursor: 指向實際LCD上的pixel座標
		- Screen cursor: 指向Screen memory的pixel座標
		- Text mode: 
			- 若當前的drawing cursor位於[60, 120], 且[HOFS, VOFS] = [10, 10],  則Screen cursor即為[60 + 10, 120 + 10] = [70, 130]
			- 由上述流程可以得知Screen cursor有可能會超出Screen的範圍(常見於256 * 256的狀況)，因此我們必須要再做mod(eg. [cursor.x, cursor.y] = [cursor.x % SCREEN_W, cursor.y % SCREEN_H])
			- 確定正確的screen cursor座標後，我們要再進一步將其轉換成正確的screen memory offset，才能存取正確的screen data
				- 由於每一個character都是8 * 8，所以我們先將screen cursor都除8(>> 3)，計算出當前的screen cursor位於哪一個character中，我們將此xy稱為**screen_data_cursor**
				- 接下來根據當前的SCREEN_W計算出screen的每一row會有幾個character，我們稱它為**charPerRow**，算法就是SCREEN_W / 8
				- 要計算出screen memory offset，公式為
				 > (charPerRow * screen_data_cursor.y + screen_data_cursor.x) * 2
				- 乘2是因為text mode的每一個screen data都是2 byte
			- 在計算完screen memory offset後，我們需要將此offset加到base address上才會是正確的screen data address
				- screen data在VRAM中以0x4000 byte為分割單位，因此想要計算base address必須先從BGCNT[08:12]讀取Screen Base Block，在乘上0x4000，最後加上VRAM base addr[0x0600'0000]，就會是base address
				- 雖然每一個Screen base block是0x4000bytes，但還是會有算上offset後，位置跑到另外一個block的狀況發生，此乃正常現象，切勿驚慌

	4. 確認當前的drawing cursor位於character的中的位置，圖取該位置的pixel data
	5. 將該palette index寫入BG row buffer，結束繪製流程
我們可以重點留意以下數點資訊:

- 僅有Mode 1, 2可支援旋轉/縮放
- 各Mode所能夠顯示的Screen數量與尺寸並不相同(詳見[Screen memory](#Screen-memory))
- 調色盤的格式可分為16*16以及256*1(詳見[Palette memory](#Palette-memory))

此模式會利用預先準備好的Character來組合出各BG圖層的內容，此模式的工作效率較bitmap模式來得高，因此大部分的商業遊戲都會使用此模式進行遊戲畫面繪製

## Screen memory

在 Character mode 下，每一個BG圖層要由那些character組成，character要在那裡出現都是由Screen memory決定

以最一般的情況[256*256]

## Palette memory

<aside class="notice">
character我們可以把他理解成一個大小為8*8的小圖塊，遊戲程式會在需要的時候將這些character搬移進VRAM中進行繪製，詳細請參閱<a href="#Character-drawing">Character drawing</a>
</aside>