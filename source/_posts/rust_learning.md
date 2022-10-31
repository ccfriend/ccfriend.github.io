---
title: rust 学习
date: 2022-10-31 09:30:21
categories: lang
tags: [rust]
---
# rust 学习

## rust 学习资源
官方参考：
- Rust 官网：https://www.rust-lang.org/
- Rust 发布版本查阅：https://releases.rs/
- Rust 语言参考：[https://doc.rust-lang.org/reference/](https://doc.rust-lang.org/reference/)
- 官方 Rust The Book: https://doc.rust-lang.org/book/

其他总结：
- Rust 语言之旅：[https://tourofrust.com/TOC_zh-cn.html](https://tourofrust.com/TOC_zh-cn.html)
- Ferrocene 语言规范：[https://spec.ferrocene.dev/](https://spec.ferrocene.dev/)
- Rust 语言圣经：[https://course.rs/about-book.html](https://course.rs/about-book.html)

## rust 工具和相关生态
- Crates 仓库： https://crates.io/
- Cargo 工具
	- https://github.com/rust-lang/cargo
	- https://doc.rust-lang.org/cargo/
- cargo 插件介绍
	- cargo-clippy: https://github.com/rust-lang/rust-clippy
	- cargo-vet: https://mozilla.github.io/cargo-vet/
-  Rust 语言 、 Rust for linux 和 wasmtime / tauri /slint 开源项目：
	- https://github.com/rust-lang/rust
	- https://github.com/Rust-for-Linux/linux
	- https://github.com/bytecodealliance/wasmtime
	- https://github.com/tauri-apps/tauri
	- https://github.com/slint-ui/slint

## 本地开发环境
编辑器/IDE 推荐
- vscode + rust-analyzer插件 + copilot（可选）
- clion
- helix（开源类 vim ，rust实现）+ ra插件
Rust 镜像
- 字节跳动镜像源 https://rsproxy.cn/

## rust在线调测工具
- Rust playground 支持单个 Rust 文件，可以方便地在线练习 Rust 语法 和 共享代码进行交流： https://play.rust-lang.org/
- Replit 支持cargo 和 crate，功能较为丰富。https://replit.com/languages/rust 
- Godbolt，支持 Rust ，可以查看编译后汇编。https://godbolt.org/

## rust测试
- Rust by example: https://doc.rust-lang.org/rust-by-example/
- Rustlings (Rust 知识检测)：[https://github.com/rust-lang/rustlings](https://github.com/rust-lang/rustlings)
- Rust Quiz   (Rust 知识检测)  https://github.com/dtolnay/rust-quiz

## 设计原则
### 面向用户
- 可靠性 Reliable
	- 如果编译成功，就可以正常工作
	- 类型安全
	- 不隐式转换
- 高性能 Performant
	 - 既高效又能保证性能，**零成本抽象**
	 - 所有权，内存安全的保证
	 - 迭代器
	 - 闭包
	 - 异步
	 - 模块性问题
	 - 动态分发？
- 支持性
	 - 语言、工具、社区支持
	 - 错误提示、修正很完善
- 生产力
	 - 和高性能是有冲突的
- 透明性
	 - 暴露底层的细节
- 多样性
	 - 嵌入式等领域

### 面向社区
- 信任和委托
- 贡献

### 金发姑娘原则
适合的才是最好的。*金发姑娘和三只熊的故事，选择了不冷不热的粥，不软不硬的椅子, 不大不小的床*
