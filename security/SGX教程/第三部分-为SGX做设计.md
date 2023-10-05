
> 这部分主要是SGX的设计，但是也涉及到windows的代码，不是linux上的。思路也是一样的


## 为enclaves部分的程序做设计

这是我们在设计Intel SGX的教程密码管理器时要遵循的一般方法。

- 识别应用程序的秘密。
- 识别这些秘密的提供者和消费者。
- 确定enclave的边界。
- 为 "enclave"定制应用程序的组件。

### 鉴别应用的秘密

为Intel SGX设计一个应用程序的第一步是确定应用程序的秘密。

秘密是指任何不打算被他人知道或看到的东西。只有用户或为其设计的应用程序才能接触到秘密，而且不应暴露给其他用户或应用程序，无论其特权级别如何。潜在的秘密可以包括财务信息、医疗记录、个人身份信息、身份数据、特许媒体内容、密码和加密密钥。

在《教程》的密码管理器中，有几个项目是可以立即识别为秘密的，如表1所示。

![[Pasted image 20221214140342.png]]

这些是显而易见的选择，但我们要扩大这个列表，包括用户的所有账户信息，而不仅仅是他们的登录信息。修订后的列表如表2所示。

![[Pasted image 20221214140700.png]]


即使不透露密码，账户信息（如服务名称和URL）对攻击者来说也是有价值的。在密码管理器中暴露这些数据会向那些有恶意的人泄露宝贵的线索。有了这些数据，他们可以选择对服务本身发起攻击，也许使用社会工程或密码重置攻击，以获得对所有者账户的访问，因为他们确切地知道目标是谁。

### 鉴别应用秘密的提供者和消费者

一旦应用程序的秘密被确定，下一步就是确定它们的来源和目的地。

在当前版本的英特尔SGX中，"enclave "代码没有加密，这意味着任何能够访问应用程序文件的人都可以拆解和检查它。根据定义，如果某样东西可以被检查到，那么它就不可能是秘密，这意味着秘密不应该被静态地编译到enclave代码中。一个应用程序的秘密必须来自其enclave之外，并在运行时加载到enclave中。在英特尔SGX术语中，这被称为向enclave提供秘密。

当秘密来自于可信计算基地（TCB）之外的组件时，重要的是要尽量减少它对不受信任的代码的暴露。(远程验证(remote attestation)是英特尔SGX的一个重要组成部分，主要原因之一是它允许服务提供商与英特尔SGX应用程序建立信任关系，然后得出一个加密密钥，可用于向应用程序提供加密的秘密，只有该客户系统上受信任的enclave可以解密）。当秘密被输出到enclave之外时，必须采取类似的谨慎措施。作为一般规则，应用程序的秘密如果没有在 "enclave "内部进行加密，就不应该被发送到不受信任的代码中。

不幸的是，对于Tutorial Password Manager应用程序，我们确实需要将秘密送入和送出 "enclave"，而这些秘密在某些时候必须以明文形式存在。最终用户将通过键盘或触摸屏输入他或她的账户信息和密码，并在将来需要的时候调用它。他们的账户密码将需要显示在屏幕上，甚至根据要求复制到Windows剪贴板上。这些都是密码管理器应用的核心要求，以便发挥作用。

这对我们来说意味着，我们不能完全消除攻击面：我们只能将其最小化，而且我们需要一些缓解策略来处理秘密，当它们以纯文本形式存在于enclave之外时。

![[Pasted image 20221214141125.png]]


## 确定enclave的边界

一旦确定了秘密，就该确定enclave的边界了。首先看一下秘密通过应用程序的核心组件的数据流。那么enclave的边界应该：

- 涵盖对你的应用程序的秘密采取行动的最小的关键组件集。
- 在可行的情况下，完全包含尽可能多的秘密。
- 尽量减少与不受信任的代码的交互和依赖。

教程密码管理器应用程序的数据流和所选择的enclave边界如图1所示。

![[Pasted image 20221214141718.png]]

这里，应用程序的秘密被描绘成圆圈，蓝色的圆圈代表在应用程序执行过程中的某个时刻以纯文本（未加密）存在的秘密，绿色的圆圈代表被应用程序加密的秘密。在加密和解密程序、密钥推导函数（KDF）和随机数生成器周围绘制了enclave边界。这为我们做了几件事。

- 用于加密我们应用程序的一些秘密（账户信息和密码）的数据库/保险箱密钥，是在enclave内生成的，并且永远不会以明文形式发送到enclave之外。
- master key来自于enclave内的用户密码，并用于加密和解密数据库/保险箱密钥（primary key）。master key是短暂的，不会以任何形式发送到飞地之外。
- 数据库/金库钥匙、账户信息和账户密码在enclave部使用加密密钥进行加密，不受信任的代码不可见（见#1和#2）。

不幸的是，我们有未加密的秘密跨越enclave界的问题，我们根本无法避免。在Tutorial Password Manager的执行过程中，用户将不得不在键盘上输入密码或将密码复制到Windows剪贴板上的某个时刻。这些都是不安全的通道，不能放在enclave内，而这些操作对于应用程序的运行是绝对必要的。这有可能是一个巨大的问题，而决定将应用程序建立在一个可管理的代码库之上，则使这个问题更加复杂。

### 保护在enclave之外的秘密

对于保护enclave外未加密的秘密，没有完整的解决方案，只有减少攻击面的缓解策略。我们能做的最好的事情是尽量减少这些信息以容易被破坏的形式存在的时间。

这里有一些关于在不受信任的代码中处理敏感数据的一般建议。

- 当你用完你的数据缓冲区时，将其清零。确保使用诸如SecureZeroMemory（Windows）和memzero_explicit（Linux）这样的函数，保证不被编译器优化掉。
- 不要使用C++标准模板库（STL）容器来存储敏感数据。STL容器有自己的内存管理，这就很难保证分配给对象的内存在对象被删除时被安全地擦除。(通过使用自定义分配器，你可以解决一些容器的这个问题）。
- 当使用托管代码（如.NET）或具有自动内存管理功能的语言时，请使用专门设计用于保存安全数据的存储类型。其他存储类型受垃圾收集器和即时编译的支配，可能不会按要求清除或释放（如果有的话）。
- 如果你必须把数据放在剪贴板上，请确保在很短的时间内清除它。特别是，不要让它在应用程序退出后还留在那里。

对于教程中的密码管理器项目，我们必须同时使用本地和托管代码。在本地代码中，我们将分配wchar_t和char缓冲区，并在释放它们之前使用SecureZeroMemory把它们擦干净。在托管代码空间中，我们将使用.NET的SecureString类。

当向非托管代码发送SecureString时，我们将使用System::Runtime::InteropServices的辅助函数来处理数据。

```cpp
using namespace System::Runtime::InteropServices;

LPWSTR PasswordManagerCore::M_SecureString_to_LPWSTR(SecureString ^ss)
{
	IntPtr wsp= IntPtr::Zero;

	if (!ss) return NULL;

	wsp = Marshal::SecureStringToGlobalAllocUnicode(ss);
	return (wchar_t *) wsp.ToPointer();
}
```

当在另一个方向上进行数据编组时，从本地代码到托管代码，我们有两个方法。如果SecureString对象已经存在，我们将使用Clear和AppendChar方法来设置来自wchar_t字符串的新值。

```cpp
password->Clear();
for (int i = 0; i < wpass_len; ++i) password->AppendChar(wpass[i]);
```

当创建一个新的SecureString对象时，我们将使用构造函数形式，从现有的wchar_t字符串创建一个SecureString。

```cpp
try {
	name = gcnew SecureString(wname, (int) wcslen(wname));
	login = gcnew SecureString(wlogin, (int) wcslen(wlogin));
	url = gcnew SecureString(wurl, (int) wcslen(wurl));
}
catch (...) {
	rv = NL_STATUS_ALLOC;
}
```

我们的密码管理器还支持将密码转移到Windows剪贴板上。剪贴板是一个不安全的存储空间，有可能被其他用户访问，因此微软建议不要将敏感数据放在上面。不过，密码管理器的意义在于使用户有可能创建他们不必记住的强大密码。它还可以创建由随机生成的字符组成的冗长密码，而这些字符是很难用手输入的。剪贴板提供了非常需要的便利，但也换来了一定程度的风险。

为了减少这种风险，我们需要采取一些额外的预防措施。首先是确保当应用程序退出时，剪贴板被清空。这是在我们的一个本地对象的析构器中完成的。

```cpp
PasswordManagerCoreNative::~PasswordManagerCoreNative(void)
{
	if (!OpenClipboard(NULL)) return;
	EmptyClipboard();
	CloseClipboard();
}
```

我们还将设置一个剪贴板定时器。当一个密码被复制到剪贴板上时，设置一个15秒的计时器，并在它启动时执行一个函数来清除剪贴板。如果一个计时器已经在运行，意味着在旧的密码过期之前，有一个新的密码被放在剪贴板上，那么这个计时器就会被取消，新的密码就会被取代。

```cpp
void PasswordManagerCoreNative::start_clipboard_timer()
{
	// Use the default Timer Queue

	// Stop any existing timer
	if (timer != NULL) DeleteTimerQueueTimer(NULL, timer, NULL);

	// Start a new timer
	if (!CreateTimerQueueTimer(&timer, NULL, (WAITORTIMERCALLBACK)clear_clipboard_proc,
		NULL, CLIPBOARD_CLEAR_SECS * 1000, 0, 0)) return;
}

static void CALLBACK clear_clipboard_proc(PVOID param, BOOLEAN fired)
{
	if (!OpenClipboard(NULL)) return;
	EmptyClipboard();
	CloseClipboard();
}
```

## 为enclave定制应用组件

随着秘密的确定和 "enclave "边界的划定，是时候在考虑到 "enclave "的同时构建应用程序了。对于在 "enclave "内可以做什么有很大的限制，这些限制将规定哪些组件在 "enclave"内，哪些在 "enclave "外，当移植一个现有的应用程序时，哪些组件可能需要一分为二。

影响到Tutorial Password Manager的最大限制是，enclave不能进行任何I/O操作。enclave不能从键盘上读取或写到显示器上，所以我们所有的秘密--密码和账户信息--必须被调入和调出enclave。它也不能从保险库文件读取或写入：解析保险库文件的组件必须与执行物理I/O的组件分开。这意味着，我们必须跨越enclave边界，不仅仅是调集我们的秘密：我们还必须调集文件内容。

![[Pasted image 20221214144249.png]]


接下来主要就是windows上的各种代码了，就不继续了，给出链接参考

- 第四部分 [Intel® Software Guard Extensions Part 4: Design an Enclave](https://www.intel.com/content/www/us/en/developer/articles/training/software-guard-extensions-tutorial-series-part-4.html)
- 第五部分 [Intel® Software Guard Extensions Part 5: Enclave Development](https://www.intel.com/content/www/us/en/developer/articles/training/intel-software-guard-extensions-tutorial-part-5-enclave-development.html)
- 第六部分 https://www.intel.com/content/www/us/en/develop/articles/intel-software-guard-extensions-tutorial-series-part-6-dual-code-paths.html
- 第七部分 https://software.intel.com/en-us/articles/intel-software-guard-extensions-tutorial-part-7-enclave-development
- 第八部分 [Intel® Software Guard Extensions Tutorial Series: Part 8, GUI...](https://www.intel.com/content/www/us/en/developer/articles/training/intel-sgx-tutorial-series-part-8-gui-integration.html)
- 第九部分 [Intel® Software Guard Extensions Tutorial Part 9: Power Events and...](https://www.intel.com/content/www/us/en/developer/articles/training/intel-sgx-tutorial-part-9-power-events-and-data-sealing.html)
-  第十部分 https://www.intel.com/content/www/us/en/develop/articles/intel-software-guard-extensions-tutorial-series-part-10-enclave-analysis-and-debugging.html 
