# 武田氏エミュータを機能強化・機能追加するパッチ

## 強化・追加内容

- デバッグ機能強化
    - デバッガの I コマンドでアドレス範囲の指定が可能
    - デバッガの O コマンドで複数バイトの指定が可能
- プログラムロード機能追加
    - MZT ファイルをエミュレータウィンドウにドロップした場合、直接メインモリにロードする
    - __【重要】__ 元々 MZT ファイルをドロップ可能なエミュレータの場合、本機能が優先され、元々の機能は無効となる

## 組み込み

### 必要なもの

- Microsoft Visual Studio Community 2022
- エミュレータのソース一式
    - [TAKEDA, toshiya's HOME PAGE](http://takeda-toshiya.my.coocan.jp/) を開く
    - Common Souce Code Project を開く
    - Download Source Code Archive (1/1/2024)  のリンクをクリックし source.7z をダウンロードする

### 準備

- source.7z の内容を適当なフォルダに展開する
- デバッグ機能強化の場合
    - 本リポジトリの src\debugger.cpp を 展開先の src\debugger.cpp に上書きする
- プログラムロード機能追加
    - 各機種共通
        - 本リポジトリの src\win32\winmain.cpp.cpp を 展開先の src\win32\winmain.cpp に上書きする
        - 本リポジトリの src\vm\vm_template.h を 展開先の src\vm\vm_template.h に上書きする
    - X1 の場合
        -  本リポジトリの src\vm\x1\x1.* を 展開先の src\vm\x1\ に上書きする
    - MZ-700 の場合
        -  本リポジトリの src\vm\mz700\mz700.* を 展開先の src\vm\mz700\ に上書きする

### ビルド（x1 の場合）

- Visual Studio で 展開先の vc++2017\x1.vcxproj を開く
- ソリューション操作の再ターゲットダイアログが開くので OK を押す ![retarget.pg](retarget.png)
- メニューから [ソリューション]-[ソリューションのビルド] を選択し、ビルドする
- 展開先の vc++2017\bin\x64\Debug\ に x1.exe ができているので、IPLROM.x1, FNT0808.x1 を用意し、起動する

__【補足】__ x1turbo, x1turboz も vcxproj が異なるだけでソースは共通なので、ビルドのみで生成可能（のはず）

### ビルド（MZ-700 の場合）

- X1 の場合との違いは次のとおり
    - vc++2017\mz700.vcxproj を使用する
    - vc++2017\x1.vcxproj\mz700.exe が実行ファイルとなる
    - MZ-700 の IPL, フォントは別途用意が必要

## 利用方法

- デバッガ機能拡張
    - I コマンド
        - 単一アドレスを指定した場合は従来通りの動作
        - アドレス範囲を指定すると、D コマンドと同様に表示される
    - O コマンド
        - 複数のバイトデータを指定可能
    - 実行例
![debugger.png](debugger.png)
- プログラムロード機能追加
    - エクスプローラから MZT ファイルを Drag&Drop する
    - 正しくロードされた場合、ウィンドウタイトルにファイル名、ロードアドレスが表示される
![load.png](load.png)

## X1/MZ-700 以外のエミュレータへの適用

- デバッグ機能拡張は機種共通部を修正しているため、自動的に反映される
- プログラムロード機能追加は、共通部（winmain.cpp, vm_template.h）の他に機種固有部の修正が必要となるが、おおよそ次の手順で可能なはず
    - src\vm\機種\機種.h の VM class の定義に load_mzt_into_memory の定義を追加
    - src\vm\機種\機種.cpp の末尾に VM::load_mzt_into_memory の実装を追加

__機種.h__
```cpp
class VM : public VM_TEMPLATE
{
    (中略)
	uint32_t VM::load_mzt_into_memory(const _TCHAR* file_path);
	// ----------------------------------------
	// for each device
	// ----------------------------------------
	
	// devices
	DEVICE* get_device(int id);
//	DEVICE* dummy;
//	DEVICE* first_device;
//	DEVICE* last_device;
};
```

__機種.cpp__

```cpp
uint32_t VM::load_mzt_into_memory(const _TCHAR* file_path)
{
	const uint32_t LOAD_ERROR = 0xffffffff;

	FILEIO* fio = new FILEIO();

	if (!fio->Fopen(file_path, FILEIO_READ_BINARY)) {
		return LOAD_ERROR;
	}

	unsigned char header[0x80];
	if (fio->Fread(header, sizeof(header), 1) != 1) {
		fio->Fclose();
		return LOAD_ERROR;
	}

	uint32_t data_size = *(uint16_t*)(header + 0x12);
	if (data_size == 0) {
		fio->Fclose();
		return LOAD_ERROR; // no data in mzt
	}
	uint32_t load_addr = *(uint16_t*)(header + 0x14);

	unsigned char *buffer = (unsigned char*)malloc(data_size);

	size_t count =  fio->Fread(buffer, data_size, 1);
	fio->Fclose();

	if (count != 1) {
		free(buffer);
		return LOAD_ERROR;
	}

	uint32_t addr = load_addr;
	unsigned char *p = buffer;
	for (int i = 0; i < data_size; i++) {
		memory->write_data8(addr++, *p++);
	}
	free(buffer);
	return load_addr;
}
```

## 保障・ライセンス

- 本パッチは無保証です
- 本パッチで修正した箇所の著作権は放棄します
- ライセンスは武田氏のライセンスに従います
