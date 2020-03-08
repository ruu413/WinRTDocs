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

### WinRT型について
WinRT内のクラスは全てWindows::Foundation::IInspectableを継承したクラスとして扱われる。基本的にXAMLから呼び出したりXamlコントロールのソースとする場合などはWinRT型のクラスとする必要がある。WinRT型にするには.idlの拡張子のMIDLでインターフェースを定義する、

MainPage.idl
~~~
namespace ZBatch
{
    runtimeclass ExplorerItemTemplateSelector : Windows.UI.Xaml.Controls.DataTemplateSelector
    //C++likeにクラスを継承できる。ここで継承できるのはWinRT型
    //::ではなく.で名前空間を指定する
    {
        ExplorerItemTemplateSelector();
        //コンストラクタ　引数なしのみしか置けない？
        Windows.UI.Xaml.DataTemplate SelectTemplateCore(IInspectable item, Windows.UI.Xaml.DependencyObject dp);
        Windows.UI.Xaml.DataTemplate SelectTemplateCore(IInspectable item);
        //メンバ関数の定義
        Windows.UI.Xaml.DataTemplate FileTemplate;
        Windows.UI.Xaml.DataTemplate FolderTemplate;
        //メンバ変数の定義　実際はGetterとSetterが用意される
    }
}
~~~
これをビルドするとGenerated Files/sources/以下にクラス名に対応した.cppと.hファイルが生成される、

Generated Files/sources/ExplorerItemSelector.h
~~~
#pragma once
#include "ExplorerItemTemplateSelector.g.h"

// Note: Remove this static_assert after copying these generated source files to your project.
// This assertion exists to avoid compiling these generated source files directly.
static_assert(false, "Do not compile generated C++/WinRT source files directly");

namespace winrt::ZBatch::implementation
{
    struct ExplorerItemTemplateSelector : ExplorerItemTemplateSelectorT<ExplorerItemTemplateSelector>
    {
        ExplorerItemTemplateSelector() = default;

        Windows::UI::Xaml::DataTemplate SelectTemplateCore(Windows::Foundation::IInspectable const& item, Windows::UI::Xaml::DependencyObject const& dp);
        Windows::UI::Xaml::DataTemplate SelectTemplateCore(Windows::Foundation::IInspectable const& item);
        Windows::UI::Xaml::DataTemplate FileTemplate();
        void FileTemplate(Windows::UI::Xaml::DataTemplate const& value);
        Windows::UI::Xaml::DataTemplate FolderTemplate();
        void FolderTemplate(Windows::UI::Xaml::DataTemplate const& value);
    };
}
namespace winrt::ZBatch::factory_implementation
{
    struct ExplorerItemTemplateSelector : ExplorerItemTemplateSelectorT<ExplorerItemTemplateSelector, implementation::ExplorerItemTemplateSelector>
    {
    };
}

~~~

Generated Files/sources/ExplorerItemSelector.cpp
~~~
#include "pch.h"
#include "ExplorerItemTemplateSelector.h"
#include "ExplorerItemTemplateSelector.g.cpp"

// Note: Remove this static_assert after copying these generated source files to your project.
// This assertion exists to avoid compiling these generated source files directly.
static_assert(false, "Do not compile generated C++/WinRT source files directly");

namespace winrt::ZBatch::implementation
{
    Windows::UI::Xaml::DataTemplate ExplorerItemTemplateSelector::SelectTemplateCore(Windows::Foundation::IInspectable const& item, Windows::UI::Xaml::DependencyObject const& dp)
    {
        throw hresult_not_implemented();
    }
    Windows::UI::Xaml::DataTemplate ExplorerItemTemplateSelector::SelectTemplateCore(Windows::Foundation::IInspectable const& item)
    {
        throw hresult_not_implemented();
    }
    Windows::UI::Xaml::DataTemplate ExplorerItemTemplateSelector::FileTemplate()
    {
        throw hresult_not_implemented();
    }
    void ExplorerItemTemplateSelector::FileTemplate(Windows::UI::Xaml::DataTemplate const& value)
    {
        throw hresult_not_implemented();
    }
    Windows::UI::Xaml::DataTemplate ExplorerItemTemplateSelector::FolderTemplate()
    {
        throw hresult_not_implemented();
    }
    void ExplorerItemTemplateSelector::FolderTemplate(Windows::UI::Xaml::DataTemplate const& value)
    {
        throw hresult_not_implemented();
    }
}

~~~

その後プロジェクトに新規ファイルとしてクラスを追加後、生成されたファイルからコピペする。

実装は全て未実装なので自分で実装する。また、MIDLに書いたインスタンス変数は実体はGetterとSetterなのでヘッダファイルにメンバ変数として定義する。ファイル先頭のstatic_assert(false, "Do not compile generated C++/WinRT source files directly");は消去する。

### TreeViewについて
ツリー表示させるTreeViewについて
自分でノードを構築する方法とデータソースからノードを作成させる2種類の方法がある。

自分でノードを構築する場合は次のようになる
~~~

            <TreeView Name="treeView">
                <TreeView.ItemTemplate>
                    <DataTemplate>
                        <TextBlock Text="{Binding Content}"/>
                    </DataTemplate>
                </TreeView.ItemTemplate>
                <TreeView.RootNodes>
                    <TreeViewNode Content="a">
                        <TreeViewNode.Children>
                            <TreeViewNode Content="aaa"/>
                            <TreeViewNode Content="abb"/>
                        </TreeViewNode.Children>
                    </TreeViewNode>
                    <TreeViewNode Content="b">
                        <TreeViewNode.Children>
                            <TreeViewNode Content="bbb"/>
                        </TreeViewNode.Children>
                    </TreeViewNode>
                </TreeView.RootNodes>
            </TreeView>
~~~
TreeView.RootNodesの内部にTreeViewNodeを入れ子状入れていく。TreeView.RootNodesやTreeViewNode.ChildrenはIVector<TreeViewNode>なのでTreeViewNode以外を入れることはできない。実際に表示する内容についてはTreeViewNode.Contentの内部に入れる。ContentはIInspectableなのでWinRT型の好きな値が入る。表示はTreeView.ItemTemplateに指定したDataTemplateを用いて表示される。何も指定しない場合to_stringが呼ばれ既定だとクラス名が表示される。

動的に要素を追加するときは次の通り

~~~
TreeViewNode a,b;
a.Content(L"a");
b.Content(L"b");
treeView().RootNodes().Append(a);
a.Children().Append(b);
//treeView()のNameを持つTreeViewが既に配置されている仮定
~~~

### WinRTのコンテナ型について
Windows::Foundation::Collections名前空間の中にいくつかの種類のコンテナが存在する。それぞれの初期化方法は次のようになる。
https://docs.microsoft.com/ja-jp/windows/uwp/cpp-and-winrt-apis/collections
~~~

Windows::Foundation::Collections::IVector<hstring> a{ single_threaded_vector<hstring>() };
Windows::Foundation::Collections::IMap<int, hstring> a{ single_threaded_map<int,hstring>() };
//KEY,VALUEのペアの型を指定する
Windows::Foundation::Collections::IObservableVector<hstring> a{ single_threaded_observable_vector<IInspectable>() };
Windows::Foundation::Collections::IObservableMap<int, IInspectable> a{ single_threaded_obserbalemap<int,IInspectalbe>() };
~~~

INotifyPropertyChangedを継承したクラスの実装