# -*- python -*-

# Big thanks to Mike Jarvis for help with the configuration prescriptions below.

import os
import sys

def CheckPython(context):
    python_source_file = """
#include "Python.h"
int main()
{
  Py_Initialize();
  Py_Finalize();
  return 0;
}
"""
    context.Message('Checking if we can build against Python... ')
    try:
        import distutils.sysconfig
    except ImportError:
        context.Result(0)
        print 'Failed to import distutils.sysconfig.'
        return False
    context.env.AppendUnique(CPPPATH=[distutils.sysconfig.get_python_inc()])
    libDir = distutils.sysconfig.get_config_var("LIBDIR")
    context.env.AppendUnique(LIBPATH=[libDir])
    libfile = distutils.sysconfig.get_config_var("LIBRARY")
    import re
    match = re.search("(python.*)\.(a|so|dylib)", libfile)
    if match:
        context.env.AppendUnique(LIBS=[match.group(1)])
    flags = [f for f in " ".join(distutils.sysconfig.get_config_vars("MODLIBS", "SHLIBS")).split()
             if f != "-L"]
    context.env.MergeFlags(" ".join(flags))
    result, output = context.TryRun(python_source_file,'.cpp')
    if not result and sys.platform == 'darwin':
        # Sometimes we need some extra stuff on Mac OS
        frameworkDir = libDir       # search up the libDir tree for the proper home for frameworks
        while frameworkDir and frameworkDir != "/":
            frameworkDir, d2 = os.path.split(frameworkDir)
            if d2 == "Python.framework":
                if not "Python" in os.listdir(os.path.join(frameworkDir, d2)):
                    context.Result(0)
                    print (
                        "Expected to find Python in framework directory %s, but it isn't there"
                        % frameworkDir)
                    return False
                break
        context.env.AppendUnique(LDFLAGS="-F%s" % frameworkDir)
        result, output = context.TryRun(python_source_file,'.cpp')
    if not result:
        context.Result(0)
        print "Cannot run program built with Python."
        return False
    context.Result(1)
    return True

def CheckNumPy(context):
    numpy_source_file = """
#include "Python.h"
#include "numpy/arrayobject.h"
void doImport() {
  import_array();
}
int main()
{
  int result = 0;
  Py_Initialize();
  doImport();
  if (PyErr_Occurred()) {
    result = 1;
  } else {
    npy_intp dims = 2;
    PyObject * a = PyArray_SimpleNew(1, &dims, NPY_INT);
    if (!a) result = 1;
    Py_DECREF(a);
  }
  Py_Finalize();
  return result;
}
"""
    context.Message('Checking if we can build against NumPy... ')
    try:
        import numpy
    except ImportError:
        context.Result(0)
        print 'Failed to import numpy.'
        print 'Things to try:'
        print '1) Check that the command line python (with which you probably installed numpy):'
        print '   ',
        sys.stdout.flush()
        subprocess.call('which python',shell=True)
        print '  is the same as the one used by SCons:'
        print '  ',sys.executable
        print '   If not, then you probably need to reinstall numpy with %s.' % sys.executable
        print '   Alternatively, you can reinstall SCons with your preferred python.'
        print '2) Check that if you open a python session from the command line,'
        print '   import numpy is successful there.'
        return False
    context.env.Append(CPPPATH=numpy.get_include())
    result = CheckLibs(context,[''],numpy_source_file)
    if not result:
        context.Result(0)
        print "Cannot build against NumPy."
        return False
    result, output = context.TryRun(numpy_source_file,'.cpp')
    if not result:
        context.Result(0)
        print "Cannot run program built with NumPy."
        return False
    context.Result(1)
    return True

def CheckLibs(context, try_libs, source_file):
    init_libs = context.env['LIBS']
    context.env.PrependUnique(LIBS=[try_libs])
    result = context.TryLink(source_file, '.cpp')
    if not result :
        context.env.Replace(LIBS=init_libs)
    return result

def CheckBoostPython(context):
    bp_source_file = """
#include "boost/python.hpp"
class Foo { public: Foo() {} };
int main()
{
  Py_Initialize();
  boost::python::object obj;
  boost::python::class_< Foo >("Foo", boost::python::init<>());
  Py_Finalize();
  return 0;
}
"""
    context.Message('Checking if we can build against Boost.Python... ')
    boost_prefix = GetOption("boost_prefix")
    boost_include = GetOption("boost_include")
    boost_lib = GetOption("boost_lib")
    if boost_prefix is not None:
        if boost_include is None:
            boost_include = os.path.join(boost_prefix, "include")
        if boost_lib is None:
            boost_lib = os.path.join(boost_prefix, "lib")
    if boost_include:
        context.env.AppendUnique(CPPPATH=[boost_include])
    if boost_lib:
        context.env.AppendUnique(LIBPATH=[boost_lib])
    result = (
        CheckLibs(context, [''], bp_source_file) or
        CheckLibs(context, ['boost_python'], bp_source_file) or
        CheckLibs(context, ['boost_python-mt'], bp_source_file)
        )
    if not result:
        context.Result(0)
        print "Cannot build against Boost.Python."
        return False
    result, output = context.TryRun(bp_source_file, '.cpp')
    if not result:
        context.Result(0)
        print "Cannot build against Boost.Python."
        return False
    context.Result(1)
    return True

# Setup command-line options
def setupOptions():
    AddOption("--prefix", dest="prefix", type="string", nargs=1, action="store",
              metavar="DIR", default="/usr/local", help="installation prefix")
    AddOption("--with-boost", dest="boost_prefix", type="string", nargs=1, action="store",
              metavar="DIR", default=os.environ.get("BOOST_DIR"),
              help="prefix for Boost libraries; should have 'include' and 'lib' subdirectories")
    AddOption("--with-boost-include", dest="boost_include", type="string", nargs=1, action="store",
              metavar="DIR", help="location of Boost header files")
    AddOption("--with-boost-lib", dest="boost_lib", type="string", nargs=1, action="store",
              metavar="DIR", help="location of Boost libraries")
    AddOption("--rpath", dest="custom_rpath", type="string", action="append",
              help="runtime link paths to add to libraries and executables; may be passed more than once")

def makeEnvironment():
    env = Environment()
    if os.environ.has_key("PATH"):
        env["ENV"]["PATH"] = os.environ["PATH"]
    if os.environ.has_key("LD_LIBRARY_PATH"):
        env["ENV"]["LD_LIBRARY_PATH"] = os.environ["LD_LIBRARY_PATH"]
    if os.environ.has_key("DYLD_LIBRARY_PATH"):
        env["ENV"]["DYLD_LIBRARY_PATH"] = os.environ["DYLD_LIBRARY_PATH"]
    if os.environ.has_key("PYTHONPATH"):
        env["ENV"]["PYTHONPATH"] = os.environ["PYTHONPATH"]
    custom_rpath = GetOption("custom_rpath")
    if custom_rpath is not None:
        env.AppendUnique(RPATH=custom_rpath)
    return env

def setupTargets(env, root="."):
    lib = SConscript(os.path.join(root, "libs", "numpy", "src", "SConscript"), exports='env')
    example = SConscript(os.path.join(root, "libs", "numpy", "example", "SConscript"), exports='env')
    test = SConscript(os.path.join(root, "libs", "numpy", "test", "SConscript"), exports='env')
    prefix = Dir(GetOption("prefix")).abspath

    env.Alias("install", env.Install(os.path.join(prefix, "lib"), lib))
    for header in ("dtype.hpp", "invoke_matching.hpp", "matrix.hpp", 
                   "ndarray.hpp", "numpy_object_mgr_traits.hpp",
                   "scalars.hpp", "ufunc.hpp",):
        env.Alias("install", env.Install(os.path.join(prefix, "include", "boost", "numpy"),
                                         os.path.join(root, "boost", "numpy", header)))
    env.Alias("install", env.Install(os.path.join(prefix, "include", "boost"),
                                     os.path.join(root, "boost", "numpy.hpp")))

checks = {"CheckPython": CheckPython, "CheckNumPy": CheckNumPy, "CheckBoostPython": CheckBoostPython}

Return("setupOptions", "makeEnvironment", "setupTargets", "checks")
