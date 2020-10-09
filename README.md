# Rule of Zero PIMPL

A single-header C++ library for PIMPLs without having to implement any special member functions as described in [oliora's blog](http://oliora.github.io/2015/12/29/pimpl-and-rule-of-zero.html).

## Code Samples

Header File:

```cpp
#include "spimpl.h"

class Copyable {
public:
  Copyable(int x);

  int x();

  // All five special members are compiler-generated.

private:
  class Impl;

  // Movable and copyable Smart PIMPL
  spimpl::impl_ptr<Impl> impl_;
};
```

Implementation File:

```cpp
Copyable::Copyable(int x)
: impl_(spimpl::make_impl<Impl>(x)) {}

class Copyable::Impl {
  Impl(int x): x(x) {}

  int x;
};

int Copyable::x() {
  return impl_->x;
}
```

## GDB Support

For easier debugging with GDB you might want to add the following to your
`.gdbinit` file. This makes `*impl_` and `impl->` behave more naturally in
GDB.

```py
python
import gdb
import gdb.xmethod
import libstdcxx.v6.xmethods
from libstdcxx.v6 import register_libstdcxx_printers

register_libstdcxx_printers (None)

class SpimplGetWorker(gdb.xmethod.XMethodWorker):
    def __init__(self, elem_type): self.elem_type = elem_type
    def get_arg_types(self): return None
    def get_result_type(self, obj): return self.elem_type.pointer()
    def __call__(self, obj):
        ptr = obj['ptr_']
        eval_string = '(*(%s*)(%s)).get()'%(ptr.type, ptr.address)
        return gdb.parse_and_eval(eval_string)

class SpimplDerefWorker(SpimplGetWorker):
    def __init__(self, elem_type): SpimplGetWorker.__init__(self, elem_type)
    def get_arg_types(self): return None
    def get_result_type(self, obj): return self.elem_type
    def __call__(self, obj):
        return SpimplGetWorker.__call__(self, obj).dereference()

class SpimplMethodsMatcher(gdb.xmethod.XMethodMatcher):
    def __init__(self):
        gdb.xmethod.XMethodMatcher.__init__(self, 'spimpl')
        self._method_dict = {
            'operator->': libstdcxx.v6.xmethods.LibStdCxxXMethod('operator->', SpimplGetWorker),
            'operator*': libstdcxx.v6.xmethods.LibStdCxxXMethod('operator*', SpimplDerefWorker),
        }
        self.methods = [self._method_dict[m] for m in self._method_dict]

    def match(self, class_type, method_name):
        if not re.match('^spimpl::impl_ptr<.*>$', class_type.tag):
            return None
        method = self._method_dict.get(method_name)
        if method is None or not method.enabled:
            return None
        return method.worker_class(class_type.template_argument(0))

gdb.xmethod.register_xmethod_matcher(None, SpimplMethodsMatcher())
end
```
