# WinRTの覚え書き

## 前準備

### おまじない
VC++17環境だとintelliSenseがエラーを起こすのでnullptr;を置き換えることで無理やりエラー回避
~~~
#if __INTELLISENSE__
#define co_await nullptr;
#endif
~~~
バグが治ったらしくintellisenseは普通に動くようになった。


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


データソースから構築する場合
再帰的なデータクラスが必要となるので礼としてフォルダー構造を持ったExplorerItemを定義する　UI側で使うのでWinRTクラスとして定義

MainPage.idl
~~~
namespace WRTApp
{
    runtimeclass ExplorerItem : Windows.UI.Xaml.Data.INotifyPropertyChanged
    {
        ExplorerItem();
        Windows.Storage.IStorageItem Item;
        Boolean IsSelected;
        String Name{ get;  };
        Windows.Foundation.Collections.IObservableVector<WRTApp.ExplorerItem> Children;
    }
}
~~~

データテンプレートをいくつか保持し、データによって使うテンプレートを切り替える
テンプレートセレクタを作成
MainPage.idl
~~~

namespace WRTApp
{
    runtimeclass ExplorerItemTemplateSelector : Windows.UI.Xaml.Controls.DataTemplateSelector
    {
        ExplorerItemTemplateSelector();
        Windows.UI.Xaml.DataTemplate SelectTemplateCore(IInspectable item, Windows.UI.Xaml.DependencyObject dp);
        Windows.UI.Xaml.DataTemplate SelectTemplateCore(IInspectable item);
        Windows.UI.Xaml.DataTemplate FileTemplate;
        Windows.UI.Xaml.DataTemplate FolderTemplate;
    }
}
~~~
テンプレートセレクタではデータテンプレートを保持する変数、
テンプレートを切り替える関数SelectTEmplateCoreを保持し、この場合ファイルとフォルダでテンプレートを切り替える
ExplorerItemTemplateSelector.cpp
~~~
    Windows::UI::Xaml::DataTemplate ExplorerItemTemplateSelector::SelectTemplateCore(Windows::Foundation::IInspectable const& item)
    {
        ExplorerItem ei = item.try_as<ExplorerItem>();
        if (ei == nullptr) {
            return nullptr;
        }
        if (ei.Item().try_as<IStorageFile>() != nullptr) {
            return m_FileTemplate;
        }
        if (ei.Item().try_as<IStorageFolder>() != nullptr) {
            return m_FolderTemplate;
        }
        return nullptr;
    }

~~~
実際のデータテンプレートはxaml上に定義(おそらくコードとしても定義できる)

MainPage.xaml
~~~
<Page.Resources>
    <DataTemplate x:Key="FolderTemplate" x:DataType="local:ExplorerItem">
         <TreeViewItem 
            ItemsSource="{x:Bind Children, Mode=TwoWay}" 
               
            HasUnrealizedChildren="True" 
            IsExpanded="{x:Bind IsExpanded, Mode=TwoWay}"
            IsSelected="{x:Bind IsSelected, Mode=TwoWay}"
            
            >
            <StackPanel Orientation="Horizontal">                    <!--Image Width="20" Source="Assets/folder.png"/-->
                <TextBlock Text="{x:Bind Name}" />
            </StackPanel>
        </TreeViewItem>
    </DataTemplate>

    <DataTemplate x:Key="FileTemplate" x:DataType="local:ExplorerItem">
        <TreeViewItem
            HasUnrealizedChildren="False" 
            IsExpanded="{x:Bind IsExpanded, Mode=TwoWay}"
            IsSelected="{x:Bind IsSelected, Mode=TwoWay}">
            <StackPanel Orientation="Horizontal">
                <!--Image Width="20" Source="Assets/file.png"/-->
                <TextBlock Text="{x:Bind Name}"/>
            </StackPanel>
        </TreeViewItem>
    </DataTemplate>

    <local:ExplorerItemTemplateSelector
        x:Key="ExplorerItemTemplateSelector"
        FolderTemplate="{StaticResource FolderTemplate}"
        FileTemplate="{StaticResource FileTemplate}" />
</Page.Resources>
~~~
テンプレートセレクタにFolderTemplateとFileTemplateを渡し、データに合ったテンプレートを使うようにする

TreeViewがテンプレートセレクタを保持することで階層的なデータ構造から適切なテンプレートを用いてTreeViewを生成できるようになる
~~~
<TreeView Name="explorerTree"
    SelectionMode="Single"
    CanReorderItems="False"
    ItemTemplateSelector="{StaticResource ExplorerItemTemplateSelector}"
    />
~~~

データソースはC++側からも渡すことができる
MainPage.cpp
~~~
StorageFolder folder = co_await picker.PickSingleFolderAsync();        
WinRTApp::ExplorerItem item;
item.Item(folder);
//フォルダを取ってExplorerITemに渡す

Collections::IObservableVector<IInspectable> vector{ winrt::single_threaded_observable_vector<IInspectable>() };
vector.Append(item);
//ExplorerItemをコンテナ型にセット
explorerTree().ItemsSource(vector);
//ItemsSourceを設定

~~~

### WinRTのコンテナ型について
Windows::Foundation::Collections名前空間の中にいくつかの種類のコンテナが存在する。それぞれの初期化方法は次のようになる。
https://docs.microsoft.com/ja-jp/windows/uwp/cpp-and-winrt-apis/collections
~~~

Windows::Foundation::Collections::IVector<hstring> a{ single_threaded_vector<hstring>() };
Windows::Foundation::Collections::IMap<int, hstring> a{ single_threaded_map<int,hstring>() };
//KEY,VALUEのペアの型を指定する
Windows::Foundation::Collections::IObservableVector<hstring> a{ single_threaded_observable_vector<IInspectable>() };
Windows::Foundation::Collections::IObservableMap<int, IInspectable> a{ single_threaded_obserbablemap<int,IInspectable>() };
~~~

### INotifyPropertyChangedを継承したクラスの実装
UI側でIObservableVector内の要素が変更されたことをコード側で検知するときはINotifyPropertyChangedを継承したクラスでイベントを発生させる必要がある
https://docs.microsoft.com/ja-jp/windows/uwp/cpp-and-winrt-apis/binding-collection
UI側で要素を使いたいのでWinRT型としてクラスを定義する
変更を検知したいプロパティはIsSelectedとしておく
MainPage.idl
~~~
namespace WinRTApp
{
    runtimeclass ExplorerItem : Windows.UI.Xaml.Data.INotifyPropertyChanged
    {
        Boolean IsSelected;
    }
    
}
~~~
その中でPropertyChangedのsetter,getterを定義することで実際に使用できる
また、インスタンス変数にイベントを持っておく
ExplorerItem.h
~~~
namespace winrt::WinRTApp::implementation
{
    struct ExplorerItem : ExplorerItemT<ExplorerItem>
    {
        winrt::event_token PropertyChanged(Windows::UI::Xaml::Data::PropertyChangedEventHandler const& handler);
        void PropertyChanged(winrt::event_token const& token) noexcept;

        void IsSelected(bool value);
        bool IsSelected();
        bool m_IsSelected;
        
        winrt::event<winrt::Windows::UI::Xaml::Data::PropertyChangedEventHandler> m_PropertyChanged;
        //イベント用変数を持っておく  
    }
}
~~~

ExplorerItem.cpp
~~~
namespace winrt::WRTApp::implementation
{
	winrt::event_token ExplorerItem::PropertyChanged(Windows::UI::Xaml::Data::PropertyChangedEventHandler const& handler)
	{
		return m_PropertyChanged.add(handler);
	}
	void ExplorerItem::PropertyChanged(winrt::event_token const& token) noexcept
	{
		m_PropertyChanged.remove(token);
	}

    
	bool ExplorerItem::IsSelected()
	{
		return m_IsSelected;
	}
	void ExplorerItem::IsSelected(bool value)
	{
		if (value == m_IsSelected) {
			return;
		}
		m_IsSelected = value;
		m_PropertyChanged(*this->get_strong(), Windows::UI::Xaml::Data::PropertyChangedEventArgs(L"IsSelected"));

	}
}
~~~
観測している変数が実際に変化した場合にのみイベントを発生させる
イベントを発生させるときは自らのポインタと何らかのIInspectableオブジェクトを
EventArgsに包んで渡すことができる

MainPage.xaml
~~~
<TreeViewItem 
    IsSelected="{x:Bind IsSelected, Mode=TwoWay}"
    >
~~~
TwoWayモードでバインドすることでC++側/UI側の両方からの情報が反映されるようになる

### ONNXモデルの実行(深層学習の学習済みモデルを使う)
用いるモデルは(10,128)のノイズを受け取り(10,3,64,64)の画像を出力するGANの学習済みモデルとする
https://docs.microsoft.com/ja-jp/windows/ai/windows-ml/get-started-desktop
~~~
StorageFile file = co_await picker.PickSingleFileAsync();
#onnxファイルを読み込み
auto model = co_await Windows::AI::MachineLearning::LearningModel::LoadFromStorageFileAsync(file);
//モデルはStorageFileのインスタンスから読み込み

LearningModelDevice device{ LearningModelDeviceKind::Default };
//モデルを実行するデバイス Cpu,Default,DirectX,DirectXMinPower,DirectXHighPerformanceが選べる

LearningModelSession session{ model, device };
//モデル,デバイスからセッションを構築


LearningModelBinding binding{ session };
//セッションに値をバインディングしていく
//バインドする値の名前は
model.InputFeatures.GetAt(0).Name()//->hstring
model.OutputFeatures.GetAt(0).Name()
//のようにして取り出せるほかhttps://www.lutzroeder.com/ai/でも確認できる

//入力はGANなのでノイズとする 入力は1次元の配列をShape付きのTensorFloatにする
std::vector<float> a(10 * 100, 0);
for (int i = 0; i < 10 * 100; ++i) {
    a[i] = std::rand()*1.0f/RAND_MAX;
}

winrt::com_array<float> array(a.begin(), a.end());
//vectorをcom_arrayに変換
binding.Bind(L"0", TensorFloat::CreateFromArray({ 10,100,1,1 },array));
//com_arrayを(10,100,1,1)の入力形状にして入力ノードにバインド

binding.Bind(L"114", TensorFloat::Create({ 10,3,64,64 }));
//空TensorFloatを出力ノードにセット

auto results = session.Evaluate(binding, L"RunId");
//実際に処理を実行する

auto resultTensor = results.Outputs().Lookup(L"114").as<TensorFloat>();
//出力を取り出す

auto resultVector = resultTensor.GetAsVectorView();
//Windows::Collection::IVectorViewを取り出す

//[-1 1]の(10,3,64,64)の画像の一枚目を[0 255]の(64,64,4)の形で取り出す チャネルにアルファを追加して0埋め
std::vector<uint8_t> image(64*64*4,0);
for (int c = 0; c < 3; ++c) {
    for (int h = 0; h < 64; ++h) {
        for (int w = 0; w < 64; ++w) {
            image[h*64*4+w*4+c] = (resultVector.GetAt(c * 64 * 64 + h * 64 + w)*0.5+0.5)*256;
        }
    }
}

//あとはストリームに流し込んで画像表示

Streams::InMemoryRandomAccessStream ras;
//書き出す対象のストリーム
Imaging::BitmapEncoder encoder = co_await Imaging::BitmapEncoder::CreateAsync(Imaging::BitmapEncoder::JpegEncoderId(), ras);
//データ形式､データを吐き出すStreamを指定
winrt::com_array image_array(image.begin(), image.end());
encoder.SetPixelData(Imaging::BitmapPixelFormat::Rgba8, Imaging::BitmapAlphaMode::Ignore, 64, 64, 1, 1, array_view<uint8_t const>{image_array.begin(), image_array.end()});
//ピクセルの形式とともにデータをいれる
//decoder,pixは上で用いた変数を仮定
co_await encoder.FlushAsync();
Windows::UI::Xaml::Media::Imaging::BitmapImage bitmap;

bitmap.SetSource(ras);

myImage().Source(bitmap);
~~~

### Windows::Graphics::Imaging::SoftwareBitmapをXamlのImageコントロールに直接表示

~~~
Windows::MediaSoftwareBitmap softwareBitmap;//SoftwareBitmapをどこからか取得
Windows::UI::Xaml::Media::Imaging::SoftwareBitmapSource source;
//co_await系以外にもUIスレッドで実行する必要があるものがあり、SoftwareBitmapSourceのコンストラクタはUIスレッドでないと例外を吐く
co_await source.SetBitmapAsync(softwareBitmap);
//sourceにbitmapを流し込み
myImage().Source(source);
//myImageがXamlのImageコントロールと仮定
~~~

### カメラから画像を取得


~~~
Windows::Media::Capture::MediaCapture mediaCapture;
auto settings = MediaCaptureInitializationSettings{};
settings.StreamingCaptureMode(StreamingCaptureMode::Video);
settings.MemoryPreference(MediaCaptureMemoryPreference::Cpu);
//CPUにするとVideoフレーム取得時に
//SoftwareBitmap,他のにするとDirect3DSurfaceが取得できる　取得形式を間違えると画像取得時にnullptr

co_await mediaCapture.InitializeAsync(settings);
//MediaCaptureを初期化


MediaProperties::ImageEncodingProperties property = MediaProperties::ImageEncodingProperties::CreateUncompressed(MediaProperties::MediaPixelFormat::Bgra8);
Capture::LowLagPhotoCapture capture = co_await mediaCapture.PrepareLowLagPhotoCaptureAsync(property);
//カメラのキャプチャを用意

auto capturedPhoto = co_await capture.CaptureAsync();
//実際にカメラから画像を取得する
auto softwareBitmap = capturedPhoto.Frame().SoftwareBitmap();
//SoftwareBitmapを取得
co_await capture.FinishAsync();


softwareBitmap = Windows::Graphics::Imaging::SoftwareBitmap::Convert(softwareBitmap, Windows::Graphics::Imaging::BitmapPixelFormat::Bgra8, Windows::Graphics::Imaging::BitmapAlphaMode::Premultiplied);
//表示できる形式に変換
co_await winrt::resume_foreground(myImage().Dispatcher());
Windows::UI::Xaml::Media::Imaging::SoftwareBitmapSource source;
//co_await系以外にもUIスレッドで実行する必要があるものがあるらしい
co_await source.SetBitmapAsync(softwareBitmap);
co_await winrt::resume_foreground(myImage().Dispatcher());
myImage().Source(source);
//表示

//TODO　この方法でのカメラの選び方

~~~

### カメラから連続してフレームを取得
asyncを使う関数はコルーチンの必要があることは注意点
~~~

auto settings = MediaCaptureInitializationSettings{};
settings.StreamingCaptureMode(StreamingCaptureMode::Video);
settings.MemoryPreference(MediaCaptureMemoryPreference::Cpu);
co_await mediaCapture.InitializeAsync(settings);
//ここまでキャプチャと変化なし

auto frameReader = co_await mediaCapture.CreateFrameReaderAsync(mediaCapture.FrameSources().First().Current().Value());
//使うフレームソースを指定　TODO 一つしかカメラが取り付けられてない場合これでも大丈夫だけどちゃんと指定した方がよい

//フレームリーダーにイベントリスナーをセットしてフレーム毎に処理する
frameReader.FrameArrived([&](MediaFrameReader sender, MediaFrameArrivedEventArgs args)->IAsyncAction {
    //ラムダ式もコルーチンで書ける　関数をセットする場合は{this, クラス名::関数名}を引数に
    auto frameref = sender.TryAcquireLatestFrame();
    if (frameref == nullptr) {
        co_return;
    }
    auto softwareBitmap = frameref.VideoMediaFrame().SoftwareBitmap();
    if (softwareBitmap == nullptr) {
        co_return;
    }
    //SoftwareBitmapが取得できたので後は煮るなり焼くなり


    //とりあえず毎フレーム表示する

    softwareBitmap = Windows::Graphics::Imaging::SoftwareBitmap::Convert(softwareBitmap, Windows::Graphics::Imaging::BitmapPixelFormat::Bgra8, Windows::Graphics::Imaging::BitmapAlphaMode::Premultiplied);
    
    co_await winrt::resume_foreground(myImage().Dispatcher());
    Windows::UI::Xaml::Media::Imaging::SoftwareBitmapSource source;
    //UIスレッドで実行する必要がある

    co_await source.SetBitmapAsync(softwareBitmap);
    co_await winrt::resume_foreground(myImage().Dispatcher());
    myImage().Source(source);
    co_return;
});
frameReader.StartAsync();
//最後にフレームリーダーを動かしたら動く
//止めるときは
frameReader.StopAsync();
~~~


### 並列パターンライブラリppl.h
#include <ppl.h>で非同期用ライブラリが使える
https://wrongwrong163377.hatenablog.com/entry/2018/07/02/003000
openMPよりははやいっぽい　std::futureの内部実装(?)
WinRT/C++のコルーチンと相性が良いっぽい
タスクを直接扱うことはあまりしなさそうだけどとりあえず調べる
コルーチン内で例外吐いてもコルーチンが止まるだけなので
~~~

concurrency::cancellation_token_source cts;
auto token = cts.get_token();
//トークンを確認してキャンセルを判定する
//https://docs.microsoft.com/ja-jp/cpp/parallel/concrt/cancellation-in-the-ppl?view=vs-2019	
//https://github.com/MicrosoftDocs/cpp-docs/issues/60
auto t = concurrency::create_task(
	[token]() 
    {
        //create_taskでタスクを作成すると同時に実行することができる
		while (true) {
			if (token.is_canceled()) {
				concurrency::cancel_current_task();
                //タスクをキャンセルしたい場合には明示的に終了させる必要がある ここではbreak;でも終了できる
			}
			else {

			}
			//co_await 1s;
			//この中でco_awaitすると挙動が怪しくなる
			return 1;
		}
	},
	token
);
//第二引数にtokenを持たせることでキャンセルが可能
auto t2 = t.then([token](concurrency::task<int> t) {
	int i = t.get();
	return i;
}, token);
//タスクが終わった時に別のタスクを実行するようにできる 引数のタスクはその終了待ちタスク
cts.cancel();
//トークンのソースでキャンセルすることでタスクをキャンセルできる
//タスクがキャンセルされた場合そのあとのタスク(この場合はt2)は実行されない(statusを確認するとcanceledになっている)

concurrency::task_status = t.wait();
//タスクがキャンセルされた場合concurrency::task_status::canceled
//正常終了はconcurrency::task_status::completed

//タスクの結果はt.get()で受け取る
//t.get()はcanceledの時concurrency::task_canceled例外を吐く
~~~

### タスクグループでタスク複数同時実行
~~~

concurrency::structured_task_group tg;
auto t1 = concurrency::make_task(
    [] 
    {
        for (int i = 0; i < 10000000000; ++i) {
	        if (concurrency::is_current_task_group_canceling()) {
		        break;
			    //concurrency::cancel_current_task()だとconcurrency::task_canceled例外になる
		    }
	    }
    }
);
auto t2 = concurrency::make_task(
    [] 
    {
	    for (int i = 0; i < 10000000000; ++i) {
		    if (concurrency::is_current_task_group_canceling()) {
			    break;
		    }
	    }
    }
);
auto t3 = concurrency::make_task(
    [] 
    {
        for (int i = 0; i < 10000000000; ++i) {
		    if (concurrency::is_current_task_group_canceling()) {
			    break;
		    }
	    }
    }
);
auto t4 = concurrency::make_task(
    [&] 
    {
	    for (int i = 0; i < 10000000000; ++i) {
		    if (i > 100) {
			    tg.cancel();
		    }
		    if (concurrency::is_current_task_group_canceling()) {
			    break;
		    }
	    }
    }
);
tg.run(t1);
tg.run(t2);
tg.run(t3);
tg.run(t4);
tg.wait();
//Concurrency::details::_Interruption_exception例外がデバッガにでるがこれはpplの内部でスタックを巻き戻すために使われているらしい
//https://social.microsoft.com/Forums/ja-JP/c8452bcd-898e-450d-8fa7-0ead8cdc2cb8/how-should-i-deal-with-these-exception?forum=winappswithnativecode
~~~


### pplの並列実行アルゴリズム(WinRTで使うときはこっちは直接使うかも)
並列に値を処理するときは当然mutexを用いる必要がある
~~~
int i = 0;
std::mutex mtx_;
//0から999までのループ 引数にはループの数値が入る
concurrency::parallel_for(0, 1000, 
    [&i,&mtx_] (int value)
    {
	    mtx_.lock();
	    i += 1;
	    mtx_.unlock();
    }
    );
//i==1000
~~~

値を全部足すみたいなことをするときはparrallel_reduce使った方がいい
~~~
std::vector<int> array{ 1,2,3,4,5 };
int sum = concurrency::parallel_reduce(array.begin(), array.end(), 0, 
    [](int acc, int i) 
    {
	    return acc + i;
    }
);
//sum==15
~~~

concurrency_for_each
並べ替えが無いときはstdのarrayやvectorなどで十分だが要素の並べ替えがある場合はconcurrency::concurrent_vectorなどを使う必要がある

とりあえずそれぞれの要素をそのまま操作する例
~~~
std::vector<int> array{ 1,2,3,4,5 };
concurrency::parallel_for_each(array.begin(), array.end(),
    [](int& i) 
	{
		i *= i;
	}
);
//array=={ 1,4,9,16,25 }
~~~
