# AppFlowy Editor 本地化修改记录

## 基本信息
- 基础版本: 5.2.0
- 修改日期: 2024-03-xx
- 托管原因: 快速输入光标异常

## 修改内容索引
1. `lib/src/editor/editor_component/service/ime/non_delta_input_service.dart`
   - 文件位置: 第 150-180 行
   - 修改内容: 优化输入防抖逻辑
     ```dart
     // 原代码
     Debounce.debounce(
       debounceKey,
       PlatformExtension.isMobile
           ? const Duration(milliseconds: 10)
           : Duration.zero,
       () async {
         // ...
       },
     );
     
     // 修改后
     Duration delayD = Duration.zero;
     if (PlatformExtension.isMobile) {
       bool isUpdate = deltas.any((e) =>
          e is TextEditingDeltaNonTextUpdate ||
          e is TextEditingDeltaReplacement);
       delayD = isUpdate ? const Duration(milliseconds: 15) : Duration.zero;
     }
     Debounce.debounce(
       debounceKey,
       delayD,
       () async {
         // ...
       },
     );
     ```
   - 修改原因: 解决移动端输入响应问题，针对非文本更新增加60ms延迟以优化性能，处理快速输入光标后多一个字母的问题

2. `lib/src/editor/editor_component/service/selection/mobile_magnifier.dart`
  - 文件位置：第43-68行
  - 修改内容：修改文字放大镜UI 
  ```dart
  class _CustomMagnifier extends StatelessWidget {
    const _CustomMagnifier({
      this.additionalFocalPointOffset = Offset.zero,
      required this.size,
    });

    final Size size;
    final Offset additionalFocalPointOffset;

    @override
    Widget build(BuildContext context) {
      return RawMagnifier(
        decoration: const MagnifierDecoration(
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.all(Radius.circular(40)),
            side: BorderSide(width: 2, color:   Color(0xff298A41)), // 增加了 border 样式
          ),
          shadows: <BoxShadow>[
            BoxShadow(
              blurRadius: 1.5,
              offset: Offset(0, 2),
              spreadRadius: 0.75,
              color: Color.fromARGB(5, 0, 0, 0),  //修改了这里的颜色 
            ),
          ],
        ),
        magnificationScale: 1.25,
        focalPointOffset: additionalFocalPointOffset,
        size: size,
        child: const ColoredBox(
          // color: Color.fromARGB(8, 158, 158, 158),
          color: Colors.transparent, // 修改了这里的颜色
          ),
        );
    }
  }
  ```
  - 修改原因：放大镜样式不一致

3. `lib/src/editor/toolbar/mobile/mobile_floating_toolbar/mobile_floating_toolbar.dart`
   - 文件位置：第 21-318 行
   - 修改内容：优化移动端浮动工具栏的显示逻辑
     ```dart
     // 原代码
     @override
     Widget build(BuildContext context) {
       return widget.child;
     }

     // 修改后
     @override
     Widget build(BuildContext context) {
       return Listener(
         onPointerUp: (event) {
           if (_isPointerMove) {
             _isPointerMove = false;
             _onScrollPositionChanged();
           }
         },
         onPointerMove: (event) {
           _isPointerMove = true;
         },
         child: NotificationListener<ScrollNotification>(
           onNotification: (notification) {
             if (notification is ScrollEndNotification && _onScrollEnd != null) {
               _onScrollEnd!.call();
               _onScrollEnd = null;
             }
             return false;
           },
           child: widget.child,
         ),
       );
     }

     // 原代码
     void _onScrollPositionChanged() {
       _toolbarContainer?.remove();
       _toolbarContainer = null;
       prevSelection = null;
     }

     // 修改后
     void _onScrollPositionChanged() {
       _toolbarContainer?.remove();
       _toolbarContainer = null;
       prevSelection = null;

       if (_isToolbarVisible && _onScrollEnd == null) {
         _onScrollEnd = () => _showAfterDelay(const Duration(milliseconds: 50));
       }
       if (widget.shrinkWrap) {
         if (_onScrollEnd != null) {
           _onScrollEnd!.call();
           _onScrollEnd = null;
         }
       }
     }

     // 原代码
     Rect _findSuitableRect(Iterable<Rect> rects) {
       assert(rects.isNotEmpty);
       final editorOffset = editorState.renderBox?.localToGlobal(Offset.zero) ?? Offset.zero;
       final minRect = rects.reduce((min, current) => min.top < current.top ? min : current);
       return minRect;
     }

     // 修改后
     Rect _findSuitableRect(Iterable<Rect> rects) {
       assert(rects.isNotEmpty);
       final editorOffset = editorState.renderBox?.localToGlobal(Offset.zero) ?? Offset.zero;
       
       // 只考虑在编辑器范围内的选区
       final rectsWithNonNegativeDy = rects.where(
         (element) => element.top >= editorOffset.dy,
       );
       if (rectsWithNonNegativeDy.isEmpty) {
         return Rect.zero;
       }

       final minRect = rectsWithNonNegativeDy.reduce((min, current) {
         if (min.top < current.top) {
           return min;
         } else if (min.top == current.top) {
           return min.top < current.top ? min : current;
         } else {
           return current;
         }
       });

       // 检查工具栏是否超出上边界
       if (minRect.top < (widget.minBarOffset?.dy ?? 0)) {
         return minRect.translate(0, (widget.minBarOffset?.dy ?? 0) - minRect.top);
       }

       // 检查工具栏是否超出下边界
       if (minRect.bottom > MediaQuery.of(widget.context).size.height - 
           MediaQuery.of(widget.context).viewInsets.bottom - 
           (widget.maxBarOffset?.dy ?? 0)) {
         return minRect.translate(
           0,
           MediaQuery.of(widget.context).size.height - 
           MediaQuery.of(widget.context).viewInsets.bottom - 
           minRect.top - 
           (widget.maxBarOffset?.dy ?? 0),
         );
       }

       return minRect;
     }
     ```
   - 修改原因：优化移动端文本选择时的工具栏显示体验，确保工具栏始终在可视区域内，并避免遮挡文本内容。主要改进包括：
     1. 优化了滚动时的工具栏处理逻辑，增加了延迟显示机制
     2. 改进了工具栏位置计算，确保不会超出屏幕边界
     3. 增加了对编辑器范围内选区的判断，避免工具栏显示在不可见区域
     4. 增加了指针移动和滚动事件的处理，优化了工具栏的显示时机

## 版本更新记录

### v5.2.0-custom.1 (2024-03-xx)
- 初始化本地化修改

### v5.2.0-custom.2 (2024-04-21)
- 优化输入防抖逻辑
  - 优化文本更新延迟逻辑
  - 解决快速输入光标异常问题
  - 修改放大器样式
  - 调整在sharkWrap模式下的mobile_floating_toolbar的定位

## 参考资料
- [相关 Issue 链接]
- [官方文档链接]
- [其他参考资料]
