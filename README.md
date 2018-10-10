rust-numpy
===========
[![Build Status](https://travis-ci.org/rust-numpy/rust-numpy.svg?branch=master)](https://travis-ci.org/rust-numpy/rust-numpy)
[![Build status](https://ci.appveyor.com/api/projects/status/bjaru43c7t1alx2x/branch/master?svg=true)](https://ci.appveyor.com/project/kngwyu/rust-numpy/branch/master)
[![Crate](http://meritbadge.herokuapp.com/numpy)](https://crates.io/crates/numpy)

Rust bindings for the NumPy C-API

API documentation
-------------
- [Latest release (possibly broken)](https://docs.rs/numpy)
- [Current Master](https://rust-numpy.github.io/rust-numpy)


Requirements
-------------
- current nightly rust (see https://github.com/PyO3/pyo3/issues/5 for nightly features, and
https://github.com/PyO3/pyo3/blob/master/build.rs for minimum required version)
- some rust libraries
  - [ndarray](https://github.com/bluss/ndarray) for rust-side matrix library
  - [pyo3](https://github.com/PyO3/pyo3) for cpython binding
  - and more (see [Cargo.toml](Cargo.toml))
- [numpy](http://www.numpy.org/) installed in your python environments (e.g., via `pip install numpy`)

**Note**
Starting from 0.3, rust-numpy migrated from rust-cpython to pyo3.
If you want to use rust-cpython, use version 0.2.1 from crates.io.


Example
---------


## Execute a Python program from Rust and get results

``` toml
[package]
name = "numpy-test"

[dependencies]
pyo3 = "^0.4.1"
numpy = "0.3"
```

``` rust
extern crate numpy;
extern crate pyo3;
use numpy::{IntoPyResult, PyArray1, get_array_module};
use pyo3::prelude::{ObjectProtocol, PyDict, PyResult, Python};

fn main() -> Result<(), ()> {
    let gil = Python::acquire_gil();
    main_(gil.python()).map_err(|e| {
        eprintln!("error! :{:?}", e);
        // we can't display python error type via ::std::fmt::Display
        // so print error here manually
        e.print_and_set_sys_last_vars(gil.python());
    })
}

fn main_<'py>(py: Python<'py>) -> PyResult<()> {
    let np = get_array_module(py)?;
    let dict = PyDict::new(py);
    dict.set_item("np", np)?;
    let pyarray: &PyArray1<i32> = py
        .eval("np.array([1, 2, 3], dtype='int32')", Some(&dict), None)?
        .extract()?;
    let slice = pyarray.as_slice()?;
    assert_eq!(slice, &[1, 2, 3]);
    Ok(())
}
```

## Write a Python module in Rust

Please see the [example](example) directory for a complete example

```toml
[lib]
name = "rust_ext"
crate-type = ["cdylib"]

[dependencies]
numpy = "0.3"
ndarray = "0.12"

[dependencies.pyo3]
version = "^0.4.1"
features = ["extension-module"]
```

```rust
extern crate ndarray;
extern crate numpy;
extern crate pyo3;

use ndarray::{ArrayD, ArrayViewD, ArrayViewMutD};
use numpy::{PyArrayDyn, ToPyArray};
use pyo3::prelude::{pymodinit, PyModule, PyResult, Python};

#[pymodinit]
fn rust_ext(_py: Python, m: &PyModule) -> PyResult<()> {
    // immutable example
    fn axpy(a: f64, x: ArrayViewD<f64>, y: ArrayViewD<f64>) -> ArrayD<f64> {
        a * &x + &y
    }

    // mutable example (no return)
    fn mult(a: f64, mut x: ArrayViewMutD<f64>) {
        x *= a;
    }

    // wrapper of `axpy`
    #[pyfn(m, "axpy")]
    fn axpy_py(
        py: Python,
        a: f64,
        x: &PyArrayDyn<f64>,
        y: &PyArrayDyn<f64>,
    ) -> PyResult<PyArrayDyn<f64>> {
        let x = x.as_array()?;
        let y = y.as_array()?;
        Ok(axpy(a, x, y).to_pyarray(py).to_owned(py))
    }

    // wrapper of `mult`
    #[pyfn(m, "mult")]
    fn mult_py(_py: Python, a: f64, x: &PyArrayDyn<f64>) -> PyResult<()> {
        let x = x.as_array_mut()?;
        mult(a, x);
        Ok(())
    }

    Ok(())
}

```

Contribution
-------------
This project is still in pre-alpha.
We need your feedback. Don't hesitate to open [issues](https://github.com/termoshtt/rust-numpy/issues)!

Version
--------
- v0.4.0(Coming soon)
  - Duplicate `PyArrayModule` and import Numpy API automatically

- v0.3.1, v0.3.2
  - Just update dependencies

- v0.3.0
  - Breaking Change: Migrated to pyo3 from rust-cpython
  - Some api addition
  - [Static type checking with PhantomData](https://github.com/rust-numpy/rust-numpy/pull/41)

- v0.2.1
  - NEW: trait `IntoPyErr`, `IntoPyResult` for error translation

- v0.2.0
  - NEW: traits `IntoPyArray`, `ToPyArray`
  - MOD: Interface of `PyArray` creation functions are changed

- v0.1.1
  - Update documents

- v0.1.0
  - First Release
  - Expose unsafe interfase of Array and UFunc API
