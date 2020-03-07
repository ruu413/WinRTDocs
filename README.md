# WinRTの覚え書き

## 前準備

### おまじない
VC++17環境だとintelliSenseがエラーを起こすのでnullptr;を置き換えることで無理やりエラー回避
~~~
#if __INTELLISENSE__
#define co_await nullptr;
#endif
~~~

~~~
using namespace winrt;
using namespace Windows::UI::Xaml;
using namespace Windows::Foundation;
using namespace Windows::Storage;
using namespace Windows::Storage::AccessCache;
using namespace Windows::Storage::Pickers;
using namespace Windows::Graphics;
~~~
あらかじめいくつかusing namespaceしておきます。

### folderをユーザーに選ばせ、フォルダを読み込み
https://docs.microsoft.com/en-us/uwp/api/windows.storage.storagefile
https://docs.microsoft.com/en-us/uwp/api/windows.storage.pickers
~~~
winrt::Windows::Storage::Pickers::FolderPicker picker;
//folderピッカーを宣言
picker.SuggestedStartLocation(PickerLocationId::HomeGroup);
//最初に開く場所を指定
//HomeGroup,Desktop,Documents,ComuputerFolder等が選べる
//https://docs.microsoft.com/en-us/uwp/api/windows.storage.pickers.pickerlocationid
picker.FileTypeFilter().ReplaceAll({ L".docx",  L".xlsx", L".pptx" });
//ファイルタイプにフィルタを設定　設定しないとエラーが出る
//ReplaceAllで全て置き換える Addでひとつづつ設定できる。
StorageFolder folder = co_await picker.PickSingleFolderAsync();
//ユーザーがフォルダを選ぶ画面が出る。
//IAsyncOperation<StorageFolder>をco_awaitで待機しunwrap
//cancelした場合nullptr
~~~

### StorageFolderの中身のファイルを取り出し
https://docs.microsoft.com/en-us/uwp/api/windows.storage.storagefile
~~~
Collections::IVectorView<StorageFile> files = co_await folder.GetFilesAsync();
//IAsyncOperation<Collections::IVectorView<StorageFile>>>
//StorageFileの配列が取り出される
//フォルダはGetFoldersAsyncで取り出す
unsigned int size = files.Size();
StorageFile& files.GetAt(0);
//ファイルをひとつづつ取り出せる
Streams::IRandomAccessStreamWithContentType f = co_await file.OpenReadAsync();
//IAsyncOperation<IRandomStreamWithContentType>をco_awaitでunwrap
//streamにデータ取り出し
~~~         

### GUIに動的に部品追加
~~~
winrt::Windows::UI::Xaml::Controls::TextBlock tbx;
//任意の部品生成
tbx.Text(L"aaaa");
//部品にパラメータ設定
addChildRoot().Children().Append(tbx);
//あらかじめNameがaddChildRootの部品がUIにあると仮定
//その子に部品を動的に追加
~~~

### ファイルからBitmapの1次元配列を取り出し

~~~
Streams::IRandomAccessStreamWithContentType f = co_await file.OpenReadAsync();
//あらかじめStorageFile fileがある前提
Graphics::Imaging::BitmapDecoder decoder = co_await Graphics::Imaging::BitmapDecoder::CreateAsync(f);
//デコーダにstreamをセット
//decoder.PixelWidth(),PixelHeight(),DpiX(),DpiY()等で画像の情報を取り出せる。
Graphics::Imaging::PixelDataProvider pixels = co_await decoder.GetPixelDataAsync();
//PixelDataの読み込み
//GetPixelDataAsync(BitmapPixelFormat,BitmapAlphaMode,BitmapTransform,ExifOrientationMode,ColorManagementMode)を使うことで取り出すPixelの形式を指定可能
winrt::com_array<uint8_t> pix = pixels.DetachPixelData();
//Pixelデータのデタッチをすることで1次元配列として取り出し
//Sizeはwidth*height*4(bgra)（デフォルト）
~~~

### Bitmapの１次元配列をjpegなどの形式でStreamに

~~~

Streams::InMemoryRandomAccessStream ras;
//書き出す対象のストリーム
Imaging::BitmapEncoder encoder = co_await Imaging::BitmapEncoder::CreateAsync(Imaging::BitmapEncoder::JpegEncoderId(), ras);
//データ形式､データを吐き出すStreamを指定

encoder.SetPixelData(Imaging::BitmapPixelFormat::Bgra8, Imaging::BitmapAlphaMode::Ignore, decoder.PixelWidth(), decoder.PixelHeight(), decoder.DpiX(), decoder.DpiY(), array_view<uint8_t const>{pix.begin(), pix.end()});
//ピクセルの形式とともにデータをいれる
//decoder,pixは上で用いた変数を仮定
co_await encoder.FlushAsync();
//FlushAsyncをしないとデータがStreamに吐かれないっぽい
                    
~~~

### 配列とIbufferの相互変換

~~~
//配列からIbuffer
winrt::com_array<uint8_t> arr(std::vector<uint8_t>(10,10));

Streams::DataWriter writer;
writer.WriteBytes(array_view<uint8_t const>{arr.begin(), arr.end()});
//writerのバッファに配列の中身を書き込み
Streams::IBuffer b = writer.DetachBuffer();
//writerのバッファを取り出す


Streams::DataReader reader = Streams::DataReader::FromBuffer(b):
//バッファをreaderにセット
reader.ReadBytes(arr);
//readerの内容を配列に読み出す。
                    
~~~
