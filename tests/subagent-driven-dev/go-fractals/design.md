# Go Fractals CLI - 设计

## 概述

一个生成 ASCII 艺术分形的命令行工具。支持两种带有可配置输出的分形类型。

## 用法

```bash
# Sierpinski 三角形
fractals sierpinski --size 32 --depth 5

# Mandelbrot 集合
fractals mandelbrot --width 80 --height 24 --iterations 100

# 自定义字符
fractals sierpinski --size 16 --char '#'

# 帮助
fractals --help
fractals sierpinski --help
```

## 命令

### `sierpinski`

使用递归细分生成 Sierpinski 三角形。

标志:
- `--size` (默认: 32) - 三角形底边的字符宽度
- `--depth` (默认: 5) - 递归深度
- `--char` (默认: '*') - 用于填充点的字符

输出: 三角形打印到 stdout，每行一行。

### `mandelbrot`

将 Mandelbrot 集合渲染为 ASCII 艺术图。将迭代计数映射到字符。

标志:
- `--width` (默认: 80) - 输出字符宽度
- `--height` (默认: 24) - 输出字符高度
- `--iterations` (默认: 100) - 逸出计算的最大迭代次数
- `--char` (默认: 梯度) - 单个字符，或省略以使用梯度 " .:-=+*#%@"

输出: 矩形打印到 stdout。

## 架构

```
cmd/
  fractals/
    main.go           # 入口点，CLI 设置
internal/
  sierpinski/
    sierpinski.go     # 算法
    sierpinski_test.go
  mandelbrot/
    mandelbrot.go     # 算法
    mandelbrot_test.go
  cli/
    root.go           # 根命令，帮助
    sierpinski.go     # Sierpinski 子命令
    mandelbrot.go     # Mandelbrot 子命令
```

## 依赖

- Go 1.21+
- `github.com/spf13/cobra` 用于 CLI

## 验收标准

1. `fractals --help` 显示用法
2. `fractals sierpinski` 输出可识别的三角形
3. `fractals mandelbrot` 输出可识别的 Mandelbrot 集合
4. `--size`, `--width`, `--height`, `--depth`, `--iterations` 标志工作
5. `--char` 自定义输出字符
6. 无效输入产生清晰的错误消息
7. 所有测试通过
