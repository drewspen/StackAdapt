# StackAdapt
GTM tag template tpl used for the base StackAdapt tag

For use when your GTM custom HTML equivalent matches the following. 

<script type="text/javascript">
  ! function(b, e, f, g, a, c, d) {
    b.saq || (a = b.saq = function() {
      a.callMethod ? a.callMethod.apply(a, arguments) : a.queue.push(arguments)
    }, b._saq || (b._saq = a), a.push = a, a.loaded = !0, a.version = "1.0", a.queue = [], c = e.createElement(f), c.async = !0, c.src = g, d = e.getElementsByTagName(f)[0], d.parentNode.insertBefore(c, d))
  }(window, document, "script", "https://tags.srv.stackadapt.com/events.js");
  saq("conv", {{StackAdapt ID PN Media RegEx Table}});
</script><noscript><img src={{StackAdapt URL}} width="1" height="1"></noscript>
