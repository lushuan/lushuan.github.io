# Python 快速上手指南 项目目录

项目目录布局
```
project_name
├── docs
│   ├── make.bat
│   ├── Makefile
│   └── source
│       ├── conf.py
│       └── index.rst
├── examples
│   └── example.py
├── project_name
│   └── package_name
│       └── __init__.py
│       ├── module1.py
│       ├── module2.py
│       ├── package1/
│       │   ├── __init__.py
│       │   ├── module3.py
│       │   ├── module4.py
│       │   └── ...
├── tests
│   └── __init__.py
├── .gitignore
├── LICENSE.txt
├── MANIFEST.in
├── README.md
├── requirements.txt
├── setup.py

```

在这个结构中：

- project_name/ 是项目的根目录。
- project_name/project_name/ 目录包含项目的源代码，__init__.py 文件是一个特殊的文件，它标志着该目录是一个 Python 包。
- module1.py 和 module2.py 是 Python 模块，它们包含 Python 代码。
- package1/ 是一个 Python 包，它可以包含其他模块和子包。包内的 __init__.py 文件用于组织包的内容。
- tests/ 目录包含项目的测试代码。
- docs/ 目录包含项目的文档。
- setup.py 是一个常见的 Python 项目配置文件，它用于安装，卸载，打包等任务。
- requirements.txt 列出了项目的依赖。

Python 项目结构并没有严格的规范，不过这种结构基本就是 Python 社区中通用的最佳实践，你可以在很多开源 Python 项目中看到这种结构。

