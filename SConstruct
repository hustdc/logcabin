import sys
import os

opts = Variables('Local.sc')

opts.AddVariables(
    ("CC", "C Compiler"),
    ("CPPPATH", "The list of directories that the C preprocessor "
                "will search for include directories", []),
    ("CXX", "C++ Compiler"),
    ("CXXFLAGS", "Options that are passed to the C++ compiler", []),
    ("LINKFLAGS", "Options that are passed to the linker", []),
    ("AS", "Assembler"),
    ("LIBPATH", "Library paths that are passed to the linker", []),
    ("LINK", "Linker"),
    ("BUILDTYPE", "Build type (RELEASE or DEBUG)", "DEBUG"),
    ("VERBOSE", "Show full build information (0 or 1)", "0"),
    ("NUMCPUS", "Number of CPUs to use for build (0 means auto).", "0"),
)

env = Environment(options = opts,
                  tools = ['default', 'protoc', 'packaging'],
                  ENV = os.environ)
Help(opts.GenerateHelpText(env))

env.Prepend(CXXFLAGS = [
    "-std=c++0x",
    "-Wall",
    "-Wextra",
    "-Wcast-align",
    "-Wcast-qual",
    "-Wconversion",
    "-Weffc++",
    "-Wformat=2",
    "-Wmissing-format-attribute",
    "-Wno-non-template-friend",
    "-Wno-unused-parameter",
    "-Woverloaded-virtual",
    "-Wwrite-strings",
    "-DSWIG", # For some unknown reason, this suppresses some definitions in
              # headers generated by protobuf 1.6 (but not 1.5) that we don't
              # use and that cause warnings with -Weffc++.
    ])

if env["BUILDTYPE"] == "DEBUG":
    env.Append(CPPFLAGS = [ "-g", "-DDEBUG" ])
elif env["BUILDTYPE"] == "RELEASE":
    env.Append(CPPFLAGS = [ "-DNDEBUG", "-O2" ])
else:
    print "Error BUILDTYPE must be RELEASE or DEBUG"
    sys.exit(-1)

if env["VERBOSE"] == "0":
    env["CCCOMSTR"] = "Compiling $SOURCE"
    env["CXXCOMSTR"] = "Compiling $SOURCE"
    env["SHCCCOMSTR"] = "Compiling $SOURCE"
    env["SHCXXCOMSTR"] = "Compiling $SOURCE"
    env["ARCOMSTR"] = "Creating library $TARGET"
    env["LINKCOMSTR"] = "Linking $TARGET"

env.Append(CPPPATH = '#')
env.Append(CPPPATH = '#/include')

# Define protocol buffers builder to simplify SConstruct files
def Protobuf(env, source):
    # First build the proto file
    cc = env.Protoc(os.path.splitext(source)[0] + '.pb.cc',
                    source,
                    PROTOCPROTOPATH = ["."],
                    PROTOCPYTHONOUTDIR = ".",
                    PROTOCOUTDIR = ".")[1]
    # Then build the resulting C++ file with no warnings
    return env.StaticObject(cc,
                            CXXFLAGS = "-std=c++0x -Ibuild")
env.AddMethod(Protobuf)

def GetNumCPUs():
    if env["NUMCPUS"] != "0":
        return int(env["NUMCPUS"])
    if os.sysconf_names.has_key("SC_NPROCESSORS_ONLN"):
        cpus = os.sysconf("SC_NPROCESSORS_ONLN")
        if isinstance(cpus, int) and cpus > 0:
            return 2*cpus
        else:
            return 2
    return 2*int(os.popen2("sysctl -n hw.ncpu")[1].read())

env.SetOption('num_jobs', GetNumCPUs())

object_files = {}
Export('object_files')

Export('env')
SConscript('Core/SConscript', variant_dir='build/Core')
SConscript('Event/SConscript', variant_dir='build/Event')
SConscript('RPC/SConscript', variant_dir='build/RPC')
SConscript('Protocol/SConscript', variant_dir='build/Protocol')
SConscript('Tree/SConscript', variant_dir='build/Tree')
SConscript('Client/SConscript', variant_dir='build/Client')
SConscript('Storage/SConscript', variant_dir='build/Storage')
SConscript('Server/SConscript', variant_dir='build/Server')
SConscript('Examples/SConscript', variant_dir='build/Examples')
SConscript('test/SConscript', variant_dir='build/test')

# This function is taken from http://www.scons.org/wiki/PhonyTargets
def PhonyTargets(env = None, **kw):
    if not env: env = DefaultEnvironment()
    for target,action in kw.items():
        env.AlwaysBuild(env.Alias(target, [], action))

PhonyTargets(check = "scripts/cpplint.py")
PhonyTargets(lint = "scripts/cpplint.py")
PhonyTargets(doc = "doxygen docs/Doxyfile")
PhonyTargets(docs = "doxygen docs/Doxyfile")
PhonyTargets(tags = "ctags -R --exclude=build --exclude=docs .")

clientlib = env.StaticLibrary("build/logcabin",
                  (object_files['Client'] +
                   object_files['Tree'] +
                   object_files['Protocol'] +
                   object_files['RPC'] +
                   object_files['Event'] +
                   object_files['Core']))
env.Default(clientlib)

daemon = env.Program("build/LogCabin",
            (["build/Server/Main.cc"] +
             object_files['Server'] +
             object_files['Storage'] +
             object_files['Tree'] +
             object_files['Protocol'] +
             object_files['RPC'] +
             object_files['Event'] +
             object_files['Core']),
            LIBS = [ "pthread", "protobuf", "rt", "cryptopp" ])
env.Default(daemon)

storageTool = env.Program("build/Storage/Tool",
            (["build/Storage/Tool.cc"] +
             [ # these proto files should maybe move into Protocol
                "build/Server/SnapshotMetadata.pb.o",
                "build/Server/Sessions.pb.o",
             ] +
             object_files['Storage'] +
             object_files['Tree'] +
             object_files['Protocol'] +
             object_files['Core']),
            LIBS = [ "pthread", "protobuf", "rt", "cryptopp" ])
env.Default(storageTool)


### scons install target

env.InstallAs('/etc/init.d/logcabin',           'scripts/logcabin-init-redhat')
env.InstallAs('/usr/bin/logcabind',             'build/LogCabin')
env.InstallAs('/usr/bin/logcabin',              'build/Examples/TreeOps')
env.InstallAs('/usr/bin/logcabin-benchmark',    'build/Examples/Benchmark')
env.InstallAs('/usr/bin/logcabin-helloworld',   'build/Examples/HelloWorld')
env.InstallAs('/usr/bin/logcabin-reconfigure',  'build/Examples/Reconfigure')
env.InstallAs('/usr/bin/logcabin-serverstats',  'build/Examples/ServerStats')
env.InstallAs('/usr/bin/logcabin-smoketest',    'build/Examples/SmokeTest')
env.InstallAs('/usr/bin/logcabin-storage',      'build/Storage/Tool')
env.Alias('install', ['/etc', '/usr'])


#### 'scons rpm' target

# monkey-patch for SCons.Tool.packaging.rpm.collectintargz, which tries to put
# way too many files into the source tarball (the source tarball should only
# contain the installed files, since we're not building it)
def decent_collectintargz(target, source, env):
    tarball = env['SOURCE_URL'].split('/')[-1]
    from SCons.Tool.packaging import src_targz
    tarball = src_targz.package(env, source=source, target=tarball,
                                PACKAGEROOT=env['PACKAGEROOT'])
    return target, tarball
import SCons.Tool.packaging.rpm as RPMPackager
RPMPackager.collectintargz = decent_collectintargz

# set the install target in the .spec file to just copy over the files that
# 'scons install' would install. Default scons behavior is to invoke scons in
# the source tarball, which doesn't make a ton of sense unless you're doing the
# build in there.
install_commands = []
for target in env.FindInstalledFiles():
    parent = target.get_dir()
    source = target.sources[0]
    install_commands.append('mkdir -p $RPM_BUILD_ROOT%s' % parent)
    install_commands.append('cp %s $RPM_BUILD_ROOT%s' % (source, target))

# We probably don't want rpm to strip binaries.
# This is kludged into the spec file.
skip_stripping_binaries_commands = [
    # The normal __os_install_post consists of:
    #    %{_rpmconfigdir}/brp-compress
    #    %{_rpmconfigdir}/brp-strip %{__strip}
    #    %{_rpmconfigdir}/brp-strip-static-archive %{__strip}
    #    %{_rpmconfigdir}/brp-strip-comment-note %{__strip} %{__objdump}
    # as shown by: rpm --showrc | grep ' __os_install_post' -A10
    #
    # brp-compress just gzips manpages, which is fine. The others are probably
    # undesirable.
    #
    # This can go anywhere in the spec file.
    '%define __os_install_post /usr/lib/rpm/brp-compress',

    # Your distro may also be configured to build -debuginfo packages by default,
    # stripping the binaries and placing their symbols there. Let's not do that
    # either.
    #
    # This has to go at the top of the spec file.
    '%global _enable_debug_package 0',
    '%global debug_package %{nil}',
]

VERSION = '0.0.1-alpha.0'
# https://fedoraproject.org/wiki/Packaging:NamingGuidelines#NonNumericRelease
RPM_VERSION = '0.0.1'
RPM_RELEASE = '0.1.alpha.0'
PACKAGEROOT = 'logcabin-%s' % RPM_VERSION

rpms=RPMPackager.package(env,
    target         = ['logcabin-%s' % RPM_VERSION],
    source         = env.FindInstalledFiles(),
    X_RPM_INSTALL  = '\n'.join(install_commands),
    PACKAGEROOT    = PACKAGEROOT,
    NAME           = 'logcabin',
    VERSION        = RPM_VERSION,
    PACKAGEVERSION = RPM_RELEASE,
    LICENSE        = 'ISC',
    SUMMARY        = 'LogCabin is clustered consensus deamon',
    X_RPM_GROUP    = ('Application/logcabin' + '\n' +
                      '\n'.join(skip_stripping_binaries_commands)),
    DESCRIPTION    =
    'LogCabin is a distributed system that provides a small amount of\n'
    'highly replicated, consistent storage. It is a reliable place for\n'
    'other distributed systems to store their core metadata and\n'
    'is helpful in solving cluster management issues. Although its key\n'
    'functionality is in place, LogCabin is not yet recommended\n'
    'for actual use.',
)

# Rename .rpm files into build/
def rename(env, target, source):
    for (t, s) in zip(target, source):
        os.rename(str(s), str(t))

# Rename files used to build .rpm files
def remove_sources(env, target, source):
    garbage = set()
    for s in source:
        garbage.update(s.sources)
        for s2 in s.sources:
            garbage.update(s2.sources)
    for g in list(garbage):
        if str(g).endswith('.spec'):
            garbage.update(g.sources)
    for g in garbage:
        if env['VERBOSE'] == '1':
            print 'rm %s' % g
        os.remove(str(g))

# Rename PACKAGEROOT directory and subdirectories (should be empty)
def remove_packageroot(env, target, source):
    if env['VERBOSE'] == '1':
        print 'rm -r %s' % PACKAGEROOT
    import shutil
    shutil.rmtree(str(PACKAGEROOT))

# Wrap cleanup around (moved) RPM targets
rpms = env.Command(['build/%s' % str(rpm) for rpm in rpms],
                   rpms,
                   [rename, remove_sources, remove_packageroot])

env.Alias('rpm', rpms)
