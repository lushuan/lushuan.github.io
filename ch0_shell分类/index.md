# shell 分类

在大部份的 UNIX 系统，三种著名且广被支持的 shell 是 Bourne shell（AT&T shell，在 Linux 下是 BASH）、C shell（Berkeley shell，在 Linux 下是 TCSH）和 Korn shell（Bourne shell 的超集）。

这三种 shell 在交谈（interactive）模式下的表现相当类似，但作为命令文件语言时，在语法和执行效率上就有些不同了。

## bash
Bourne shell 是标准的 UNIX shell，以前常被用来做为管理系统之用。大部份的系统管理命令文件，例如 rc start、stop 与 shutdown 都是 Bourne shell 的命令档，且在单一使用者模式（single user mode）下以 root 签入时它常被系统管理者使用。

Bourne shell 是由 AT&T 发展的，以简洁、快速著名。 Bourne shell 提示符号的默认值是 $。

## csh
C shell 是柏克莱大学（Berkeley）所开发的，且加入了一些新特性，如命令列历程（history）、别名（alias）、内建算术、档名完成（filename completion）、和工作控制（job control）。

对于常在交谈模式下执行 shell 的使用者而言，他们较喜爱使用 C shell；但对于系统管理者而言，则较偏好以 Bourne shell 来做命令档，因为 Bourne shell 命令档比 C shell 命令档来的简单及快速。C shell 提示符号的默认值是 %。

## ksh
Korn shell 是 Bourne shell 的超集（superset），由 AT&T 的 David Korn 所开发。它增加了一些特色，比 C shell 更为先进。Korn shell 的特色包括了可编辑的历程、别名、函式、正规表达式万用字符（regular expression wildcard）、内建算术、工作控制（job control）、共作处理（coprocessing）、和特殊的除错功能。Bourne shell 几乎和 Korn shell 完全向上兼容（upward compatible），所以在 Bourne shell 下开发的程序仍能在 Korn shell 上执行。Korn shell 提示符号的默认值也是 $。

在 Linux 系统使用的 Korn shell 叫做 pdksh，它是指 Public Domain Korn Shell。除了执行效率稍差外，Korn shell 在许多方面都比 Bourne shell 为佳；但是，若将 Korn shell 与 C shell 相比就很困难，因为二者在许多方面都各有所长，就效率和容易使用上看，Korn shell 是优于 C shell，相信许多使用者对于 C Shell 的执行效率都有负面的印象。

在 shell 的语法方面，Korn shell 是比较接近一般程序语言，而且它具有子程序的功能及提供较多的资料型态。至于 Bourne shell，它所拥有的资料型态是三种 shell 中最少的，仅提供字符串变量和布尔型态。在整体考量下 Korn shell 是三者中表现最佳者，其次为 C shell，最后才是 Bourne shell，但是在实际使用中仍有其它应列入考虑的因素，如速度是最重要的选择时，很可能应该采用 Bourne shell，因它是最基本的 shell，执行的速度最快。

## tcsh
tcsh 是近几年崛起的一个免费软件（Linux 下的 C shell 其实就是使用 tcsh）执行，它虽然不是 UNIX 的标准配备，但是从许多地方您都可以下载到它。如果您是 C shell 的拥护者，笔者建议不妨试试 tcsh，因为您至少可以将它当作是 C shell 来使用。如果您愿意花点时间学习，您还可以享受许多它新增的优越功能，例如：

- tcsh 提供了一个命令列（command line）编辑程序。
- 提供了命令列补全功能。
- 提供了拼字更正功能。它能够自动检测并且更正在命令列拼错的命令或是单字。
- 危险命令侦测并提醒的功能，避免您一个不小心执行了rm* 这种杀伤力极大的命令。
- 提供常用命令的快捷方式（shortcut）。

## ash
一个简单的轻量级的 Shell，占用资源少，适合运行于低内存环境，但是与下面讲到的 bash shell 完全兼容

## Shell分类总结

在大部份的 UNIX 系统，三种著名且广被支持的 shell 是 Bourne shell（AT&T shell，在 Linux 下是 BASH）、C shell（Berkeley shell，在 Linux 下是 TCSH）和 Korn shell（Bourne shell 的超集）。

这三种 shell 在交谈（interactive）模式下的表现相当类似，但作为命令文件语言时，在语法和执行效率上就有些不同了。
