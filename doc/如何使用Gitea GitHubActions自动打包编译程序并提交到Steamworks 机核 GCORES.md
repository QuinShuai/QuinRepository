# 如何使用Gitea/GitHubActions自动打包编译程序并提交到Steamworks | 机核 GCORES
最近在搞 Switshot 新版本，同时在弄 Steam Deck 上的伴侣 app 需要提交到 Steamworks，加上最近 Gitea 也推出了 CI/CD 功能，想着干脆就把能上 CI/CD 的东西全部上一遍好了。但比较可惜的是，中文互联网上好像对通过 CI/CD 管线提交 Steamworks 的相关问题没有太多记录，加上我的开发流程比较 tricky，于是打算先记录下来，希望可以帮到别人（顺便传教 CI/CD）。

需要注意的是，这篇文章比较注重通过 CI/CD 管线将编译好的工件提交到 Steamworks 的配置的说明。如果你想了解你开发的游戏如何通过 CI/CD 编译，可以搜搜网上其他的资料。

如果你已经对我上面说的许多名词有了基础概念，那么请直接跳过这一段；否则，这里是一些简单的解答。

CI/CD：「持续集成与持续交付（Continuous Integration and Continuous Delivery）」的简称。你可以将它视作一种高级的自动化：假设你的游戏发版需要经过编译、检查、测试、分发等一系列流程，你可以将它们集成为一组自动化动作。当你的开发工作完成到一定水平、触发这个自动化动作之后，机器会自动完成你预设的动作，开始进行编译和检查；人工完成测试确定可以发布之后，还能通过自动化提交至商店进行版本交付。

工件（Artifacts）：又译「产物」，即 CI/CD 流程中产生出来的产品/记录/文件等等等等。编译好的软件包也在其列。

Steamworks：类似于 App Store Connect，专为开发者设置的商店后台。

Gitea/GitHub Actions：Gitea 是一款自部署的、基于 git 的代码托管服务器；而 GitHub Actions 是一个代码自动化工具，可以在代码提交的时候自动触发一系列的操作，进而帮助开发者完成 CI/CD 工作。Gitea 1.19.0 版本开始新增与 GitHub Actions 对标的 Gitea Actions 功能，不仅功能相近，同时也可以（几乎不经修改地）使用 GitHub Actions 的配置文件。最大的区别在于，Gitea Actions 要求自己部署一个运行器（runner）用于编译或执行其他自动化流程，而 GitHub Actions 提供了许多免费的运行器。由于我的工程的 CI/CD 流程托管在 Gitea，因此会与你在 GitHub 使用的时候会有些出入，但二者的操作几乎可以等价。

想让机器自己学会编译、交付你的代码，首先肯定得告诉它该怎么做。这就是 Actions 流程文件的目的。

首先，来创建第一个流程文件。请在你的代码根目录创建一个 .gitea 或 .github 文件夹，然后在新文件夹中再创建 workflows 文件夹。创建一个 YAML 文件，我在这里就给它起名为 compiling.yaml（编译.yaml）。

这个配置文件其实不难理解。简单来说，首先定义这个流程名字是「GitHub Actions Demo」，然后说明中插入你的 GitHub 用户名，代表这个流程是你写的，然后设置只要 GitHub 收到所有分支中的代码推送就触发这个自动化。接着，创建一个名为「Explore-GitHub-Actions」的工作（job），并在安装了 Ubuntu 的操作系统的执行器上执行一些操作，包括打印欢迎消息、通过变量输出一些上下文信息（操作系统、触发流程的分支）、拉取代码到当前执行器中、列出目前代码中的所有文件，等等。

当你提交更改并推送到远程仓库之后，你的 GitHub/Gitea 仓库页面的 Actions 标签右边应该会多出来一个「1」的徽标，代表正在开始自动化流程。你可以随时在 Actions 页面检查自动化执行历史、正在执行的自动化的进展。

当然，我这个流程是已经改过、用于生产的流程。

编写自动化流程就是这么简单：一个 name 参数，用于备注给人看这个流程是在做什么；一个 run 参数，代表执行器在这一步应该做什么。假设你在本地需要使用 make build 终端指令来要求电脑编译你的代码，只要你在自动化工作中检出（check out）了你的代码，你就可以为 make build 指令添加一个 job steps，并在 run 参数中添加 make build，指示自动化需要执行这个指令，在这里就是开始编译的意思。

你可能已经注意到，在编译步骤之前的步骤，还有一个 uses 的参数。它的意思是使用别人已经编写好的 Action 的操作。在这里是将你的代码检出（拉取）到执行器的执行环境中。

接下来开始实际动手编写自动化。首先，我需要做的事情是，下载并应用 Node.js 环境，然后检出代码。完成后，安装依赖并开始打包编译。此时我的 CI/CD 自动化流程是这样：

由于我需要交由 CI/CD 自动化的产品并不是一款游戏，而是一款（基于 Electron 的）应用程序，因此直接拿走我的配置文件应该是没办法应用到各位的游戏项目中的（[GitHub Marketplace](https://www.gcores.com/link?target=https%3A%2F%2Fgithub.com%2Fmarketplace%3Ftype%3Dactions%26category%3Dgame-ci) 有许多针对编译打包游戏引擎的 Actions 可供选择）；但为了不让各位太过迷惑，以上信息重点用作背景信息。接下来才是重点。

CI/CD 流程启动 Electron 打包器后会将我的打包文件放在 dist 目录下。同样在 dist 目录下的还有打包日志文件 builder-debug.yml，以及被统一放在 linux-unpacked 的针对 Linux 系统的软件包的一些其他产物。我首先将它们统一打包为 zip，然后上传到 Cloudflare R2 作为存档，以便我下载编译后产物或是检查历史编译文件。

注意，在这里我们用到了变量，即使用美金符号和两个花括号包裹的东西。

首先是 secret 前缀的变量，它代表的是秘密变量/密钥变量，可以在项目设置中进行人为设定：

*   Gitea：进入项目后点击页面标签页栏右侧的「设置」，左侧目录点击「Actions」-「密钥」。
    
*   GitHub：进入项目后点击页面标签页栏的「Settings」，左侧目录点击「Secrets and variables」-「Actions」。
    

在里面新建好你设置的变量之后，在流程中就可以用 secret.YOUR\_SECRET\_NAME 的方式来访问了。秘密变量一旦新建，你和你的同事都不能再看见，只能修改，即使将它在 CI/CD 打印出来，也只会显示三个星号。

然后是 gitea 前缀的变量。这些变量代表是执行环境上下文，包括运行任务的编号等等。这些变量在 GitHub 和 Gitea 中都通用，只是前缀有所差异（即如果你使用 GitHub Actions，你应该使用 github 前缀）。[这里](https://www.gcores.com/link?target=https%3A%2F%2Fdocs.github.com%2Fzh%2Factions%2Flearn-github-actions%2Fcontexts)是所有可用的变量列表。

最终，这个步骤的效果是：将所有放置在 dist 文件夹中的工件打包成 dist.zip，并提供 Cloudflare R2 的密钥，要求执行器在存储空间中新建以任务编号为名的文件夹，并上传至这个新文件夹。当然，你不一定也要用 Cloudflare R2，AWS S3、阿里云 OSS 和腾讯云 COS 等兼容 S3 接口的存储服务都有对应的 Actions，在 GitHub Marketplace 中稍加搜索就能找到。

接下来，就需要考虑提交软件包至 Steamworks 的事情了。

首先，经过一番研究，我个人推荐专门注册一个用于 CI/CD 流程的 Steam 小号。前往 partner.steamgames.com 登录，选择「用户与权限」-「管理用户」，你会在左侧看到「邀请用户」的按钮。点击之后会有一个表单，邮箱地址需要填写没有注册过 Steam 的邮箱地址，名字可以做一下简单的备注。右侧的权限列表勾选「编辑应用元数据」、「在 Steam 上发布更改」两个。

接下来，这个新的邮箱会收到一封邀请邮件。通过邀请邮件中的链接注册一个新的小号并接受邀请，然后主号再确认一下，小号就可以用来作为 CI/CD 提交流程工具了。

然后，在自己的电脑上安装一下 SteamCMD 工具（看这篇文章的应该没有人没装过吧？），再使用命令行的方式来登录一次，这次登录会提示需要 SteamGuard 验证码——别慌，看看邮箱。

登录成功之后需要找到配置文件。macOS 上，配置文件存放在下面这个位置。

将这个文件的内容转换为 Base64，然后打开 Gitea 或 GitHub 的秘密变量配置页面，添加以下秘密变量：

*   STEAM\_CONFIG\_VDF，将上面的步骤的 Base64 粘贴进来
    
*   STEAM\_USERNAME，小号的 Steam 用户名
    

如果你在执行这些操作的电脑上有登录过桌面版 Steam，个人建议你在桌面版 Steam 注销后重新登录一下，因为这个配置文件里可能还有你主号的登录信息（这个操作不会影响小号）。

下面就是对 CI/CD 编译后的文件夹做点小手术的时候了：

*   清理不需要的文件
    
*   修改打包好的二进制文件的名字（以适应 Steam 中「启动选项」的要求）
    
*   将 dist 文件夹下的不同工件，按操作系统整理到不同的目录中
    

由于我只需要针对 Steam OS 进行开发，因此只需要将二进制包的名字改成统一的名称，然后将它塞到 dist/SteamOSBinary 文件夹下即可。注意，即使你的游戏或软件只面向一个平台，你也需要多套一个文件夹。

万事俱备，终于可以要求自动化流程将 depot 交付到 Steamworks 了：

复制我的流程的时候请注意把 appId 参数改成你的 Steamworks 应用的 appId，将 rootPath 参数设置为你的二进制输出主目录，主目录下存放不同操作系统的二进制的目录填写到 depot\[x\]Path 参数中。其他更多参数可以看[官方的文档](https://www.gcores.com/link?target=https%3A%2F%2Fgithub.com%2Fgame-ci%2Fsteam-deploy)。

另外，我额外做了一个 if 参数，代表只有当主分支接受了提交之后才执行这一操作（以避免意外地将用于内部测试的二进制交付到 Steam。）我在 buildDescription 中备注了当前工作的序号，这样在 Steamworks 后台就可以通过备注轻松查找到是由哪一次自动化工作生成的二进制版本。

编写好后，将你的代码推送到仓库中，然后等待大约十分钟到一小时不等（取决于你的游戏或者软件有多大）。接下来，看到所有流程步骤亮起绿灯之后，就可以在 Steamworks 后台找到刚才由 CI/CD 自动化交付的软件包了！

如果你需要我的完整的 Actions 配置，[可以从这里找到](https://www.gcores.com/link?target=https%3A%2F%2Fgist.github.com%2FAstrian%2F20ae8c3939855c14adcfd5c8c8fc4491)。限于篇幅和经验不足的原因，只能为大家详细讲解在 Linux 环境下的配置方法，但其他操作系统下的配置方法也大差不差，举一反三一下即可通用。也希望这篇文章可以作为抛砖引玉，让更多大佬再来讲解有关其他平台下自动化编译、打包、交付游戏二进制的方法。