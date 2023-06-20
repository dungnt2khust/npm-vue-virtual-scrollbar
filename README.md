# vue-virtual-scrollable

'''html
<!-- Use CSS Selector -->
<VueVirtualScrollbar
   :horizontal="true"
   target="#scrollable"
   @positionChanged="positionChanged"
   >
</VueVirtualScrollbar>

<!-- Use element -->
<VueVirtualScrollbar
   style="position: absolute; top: 0; bottom: 0; right: 0"
   :horizontal="false"
   :target="target"
   >
</VueVirtualScrollbar>
'''