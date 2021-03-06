project (
    'motion',
    'c',
    version: run_command('sh', '-c', 'cd "${MESON_SOURCE_ROOT}"; sh ./version.sh').stdout().strip(),
    license : 'GPL2',
    default_options : [
        'warning_level=1',
        'buildtype=plain',
    ],
)


cc = meson.get_compiler('c')
cdata = configuration_data()


add_global_arguments('-std=gnu99', '-Wall', language : 'c')
libm_dep = cc.find_library('m', required : false)


cdata.set('VERSION', meson.project_version())
cdata.set('PACKAGE_NAME', 'motion')
cdata.set('PACKAGE_VERSION', meson.project_version())
cdata.set('sysconfdir', get_option('sysconfdir'))
cdata.set('BIN_PATH', get_option('bindir'))
cdata.set('prefix', get_option('prefix'))


add_global_arguments(
    '-DVERSION="@0@"'.format(cdata.get('PACKAGE_VERSION')),
    '-Dsysconfdir="@0@"'.format(cdata.get('sysconfdir')),
    language : 'c'
)


docdir = join_paths(get_option('datadir'), 'doc', cdata.get('PACKAGE_NAME'))
examples_dir = join_paths(docdir, 'examples')


# taken from the `OBJ` variable inside of the `Makefile.in` file
SRC_FILES = [
    'alg.c',
    'conf.c',
    'draw.c',
    'event.c',
    'ffmpeg.c',
    'jpegutils.c',
    'logger.c',
    'md5.c',
    'motion.c',
    'netcam.c',
    'netcam_ftp.c',
    'netcam_http.c',
    'netcam_jpeg.c',
    'netcam_rtsp.c',
    'netcam_wget.c',
    'picture.c',
    'rotate.c',
    'stream.c',
    'track.c',
    'video_bktr.c',
    'video_common.c',
    'video_loopback.c',
    'video_v4l2.c',
    'webhttpd.c',
]


threads_dep = dependency('threads')
libjpeg_dep = dependency('libjpeg')


## BEGIN: Check for pthread_setname_np (nonstandard GNU extension)
code = '
    #include <pthread.h>
    int main(void) {
        pthread_setname_np(pthread_self(), "name");
    }
'
if cc.compiles(code)
    cdata.set('HAVE_PTHREAD_SETNAME_NP', 1)
endif
## END: Check for pthread_setname_np (nonstandard GNU extension)

## BEGIN: Check for pthread_getname_np (nonstandard GNU extension)
code = '
    #include <pthread.h>
    int main(void) {
        pthread_getname_np(pthread_self(), NULL, 0);
    }
'
if cc.compiles(code)
    cdata.set('HAVE_PTHREAD_GETNAME_NP', 1)
endif
## END: Check for pthread_getname_np (nonstandard GNU extension)

## BEGIN: Check for XSI strerror_r
code = '
    #include <string.h>
    #include <errno.h>
    int main(void) {
        char buf[1024];
        int ret = strerror_r(ENOMEM, buf, sizeof(buf));
        return ret;
    }
'
if cc.compiles(code)
    cdata.set('XSI_STRERROR_R', 1)
endif
## END: Check for XSI strerror_r


if get_option('WITH_V4L2')
    if cc.has_header('linux/videodev2.h')
        cdata.set('HAVE_LINUX_VIDEODEV2_H', 1)
        cdata.set('HAVE_V4L2', 1)
    else
        error('V4L2 support requested but not found')
    endif
endif
if get_option('WITH_BKTR')
    check_headers = []
    sys = host_machine.system().to_lower()
    if sys.contains('freebsd')
        check_headers += [ 'dev/bktr/ioctl_meteor.h', 'dev/bktr/ioctl_bt848.h' ]
    elif sys.contains('openbsd') or sys.contains('netbsd')
        check_headers += [ 'dev/ic/bt8xx.h' ]
    else
        error('BKTR support requested, but no BSD platform was recognized')
    endif

    foreach h : check_headers
        if not cc.has_header(h)
            error('BKTR support requested, but required headers were not found')
        endif
    endforeach

    cdata.set('HAVE_BKTR', 1)
endif
mmal_deps = []
if get_option('WITH_MMAL')
    mmal_deps = [
        dependency('mmal'),
        cc.find_library('vcos'),
        cc.find_library('vchostif'),
        cc.find_library('vcilcs'),
    ]
    SRC_FILES += [
        'mmalcam.c',
        'raspicam/RaspiCamControl.c',
        'raspicam/RaspiCLI.c',
    ]
    cdata.set('HAVE_MMAL', 1)
endif


ffmpeg_deps = []
if get_option('WITH_FFMPEG')
    ffmpeg_deps += [
        dependency('libavutil'),
        dependency('libavformat'),
        dependency('libavcodec'),
        dependency('libswscale'),
        dependency('libavdevice'),
    ]
    cdata.set('HAVE_FFMPEG', 1)
endif
webp_deps = []
if get_option('WITH_WEBP')
    webp_deps += dependency('libwebp')
    cdata.set('HAVE_WEBP', 1)
endif


mysql_deps = []
if get_option('WITH_MYSQL')
    mysql_deps += dependency('mysqlclient')
    cdata.set('HAVE_MYSQL', 1)
endif
postgresql_deps = []
if get_option('WITH_PGSQL')
    postgresql_deps += dependency('libpq')
    cdata.set('HAVE_PGSQL', 1)
endif
sqlite3_deps = []
if get_option('WITH_SQLITE3')
    sqlite3_deps += dependency('sqlite3')
    cdata.set('HAVE_SQLITE3', 1)
endif


check_headers = [
    'stdio.h',
    'unistd.h',
    'stdint.h',
    'fcntl.h',
    'time.h',
    'signal.h',
    'sys/ioctl.h',
    'sys/mman.h',
    'sys/param.h',
    'sys/types.h',
]
foreach h : check_headers
    if not cc.has_header(h)
        error('Required header not found: @0@'.format(h))
    endif
endforeach


# mesonify config.h.in
config_h_meson = configure_file(
    input: 'config.h.in',
    output: 'config.h.meson',
    command : [ 'sed', '-r', '-e', 's|^# *undef|#mesondefine|', '@INPUT@' ],
    capture: true,
)
configure_file(
    input: config_h_meson,
    output: 'config.h',
    configuration: cdata,
)
configinc = include_directories('.')


motion_exe = executable(
    'motion',
    sources : SRC_FILES,
    dependencies: [ libm_dep, libjpeg_dep, threads_dep ] +
        ffmpeg_deps + webp_deps + mysql_deps + postgresql_deps + sqlite3_deps + mmal_deps,
    include_directories: [ configinc ],
    install: true,
)


examples = []
examples_array = [
    [ 'motion-dist.conf.in',  'motion-dist.conf' ],
    [ 'camera1-dist.conf.in', 'motion-dist.conf' ],
    [ 'camera2-dist.conf.in', 'motion-dist.conf' ],
    [ 'camera3-dist.conf.in', 'motion-dist.conf' ],
    [ 'camera4-dist.conf.in', 'motion-dist.conf' ],
    [ 'motion.service.in',    'motion-dist.conf' ],
]
foreach e : examples_array
    examples += configure_file( input: e.get(0), output: e.get(1), configuration: cdata, install_dir: examples_dir )
endforeach


install_data(
    [
        'CHANGELOG',
        'COPYING',
        'CREDITS',
        'README.md',
        'motion_guide.html',
        'mask1.png',
        'normal.jpg',
        'outputmotion1.jpg',
        'outputnormal1.jpg',
    ],
    install_dir: docdir,
)
