AC_INIT( S3DE, 0.1.2, , s3de)

# -----------------------------
# application-specific settings

# default choices
# set each variable to 0 (false) or 1 (true)
use_parallel=1
use_shared=1

# -----------------------------

# store variables from the environment (may be set or not)
topdir=$TOPDIR
arch=$ARCH
env_cc=$CC
cpp=$CPP
cflags=$CFLAGS
cppflags=$CPPFLAGS
dflags=$DFLAGS
iflags=$IFLAGS
f77=$F77
f90=$F90
mpif90=$MPIF90
fflags=$FFLAGS
f90flags=$F90FLAGS
fautodbl=$FAUTODBL
ld=$LD
ldflags=$LDFLAGS
blas_libs=$BLAS_LIBS
lapack_libs=$LAPACK_LIBS
fft_libs=$FFT_LIBS
mpi_libs=$MPI_LIBS
mass_libs=$MASS_LIBS
libdirs=$LIBDIRS
ar=$AR
arflags=$ARFLAGS
arflags_dynamic=$ARFLAGS_DYNAMIC
includefftw=$INCLUDEFFTW
pgplot_libs=$PGPLOT_LIBS
#
iotk_integer_kinds=$IOTK_INTEGER_KINDS
iotk_logical_kinds=$IOTK_LOGICAL_KINDS
iotk_real_kinds=$IOTK_REAL_KINDS
iotk_fc=$IOTK_FC
#

# command-line arguments
for arg in $*
do
	if   test "$arg" = "--enable-parallel"  ; then use_parallel=1
	elif test "$arg" = "--disable-parallel" ; then use_parallel=0
	elif test "$arg" = "--enable-shared"    ; then use_shared=1
	elif test "$arg" = "--disable-shared"   ; then use_shared=0
	fi
done

# configure for current directory by default
if test "$topdir" = "" ; then topdir="`pwd`" ; fi

# check system type (no cross-compilation for now)
AC_CANONICAL_BUILD

# many HPC systems are configured so that running parallel programs
# interactively is disabled: on those systems, AC_PROG_F77 and AC_PROG_CC
# would fail because they can't run the compiled executables.
# to work around that, let's pretend we are cross-compiling even if we aren't
# !!! this relies on undocumented Autoconf behavior !!! 
if test "$host" = "" ; then host=$build ; fi
cross_compiling=yes

# identify architecture
if test "$arch" = ""
then
	case $host in
		ia64-*-linux-gnu )      arch=ia64   ;;
		x86_64-*-linux-gnu )    arch=amd64  ;;
		*-pc-linux-gnu )        arch=ia32   ;;
		*-ibm-aix* )            arch=aix    ;;
		mips-sgi-* )            arch=mips   ;;
		alphaev*-dec-osf* )     arch=alpha  ;;
		alphaev*-*-linux* )     arch=alinux ;;
		sparc-sun-* )           arch=sparc  ;;
		*cray-unicosmp* )       arch=crayx1 ;;
		powerpc-apple-darwin* ) arch=macppc ;;
		i686-apple-darwin* )    arch=mac686 ;;
		*-pc-cygwin )           arch=cygwin ;;
	esac
fi
echo checking architecture... $arch

# candidate archiver and archiver flags
try_ar="ar"
try_arflags="ruv"
try_arflags_dynamic="ruv"
try_iflags="-I../include"

# candidate fortran compilers good for all cases
try_mpif90="mpif90"
try_f90="g95 gfortran f90"

# candidate compilers and flags based on architecture
case $arch in
ia64 | amd64 )
	try_f90="ifort ifc efc pgf90 pathf95 $try_f90"
        try_dflags="-D__LINUX64"
	;;
ia32 )
	try_f90="ifort ifc pgf90 pathf95 $try_f90"
        try_dflags="-D__LINUX"
	;;
aix )
	try_mpif90="mpxlf90_r mpxlf90"
	try_f90="xlf90_r xlf90 $try_f90"
	try_arflags="-X64 ruv"
	try_arflags_dynamic="-X64 ruv"
	try_dflags="-D__AIX -D__XLF"
	;;
mips )
	try_dflags="-D__COMPLIB -D__ORIGIN"
	;;
alpha )
	try_dflags="-D__ALPHA"
	;;
alinux )
	try_f90="fort $try_f90"
	try_dflags="-D__ALPHA"
	;;
sparc )
	try_mpif90="mpf90"
	try_dflags="-D__SUN"
	;;
crayx1 )
	try_f90="ftn $try_f90"
	try_dflags="-D__X1"
	;;
macppc )
	try_mpif90="mpxlf90 mpf90"
	try_f90="xlf90 $try_f90"
	try_dflags="-D__XLF"
# due to bugs in xlf 8.1 for powerpc mac os-x, the iotk library must be
# created as a dynamical library, using "libtool"
  	try_ar="libtool"
 	try_arflags="-static -o"
  	try_arflags_dynamic="-dynamic -arch_only ppc -all_load -single_module -L/opt/ibmcmp/xlf/8.1/lib -lxlf90 -lxl -lc -o"
	;;
mac686 | cygwin )
	try_f90="ifort $try_f90"
	;;
* )
	AC_MSG_WARN(unsupported architecture?)
	;;
esac

# check serial Fortran 90 compiler. This must be done before performing
# the check for the parallel compiler (section below) because option
# --disable-parallel will do very strange things otherwise. The reason
# seems to be that autoconf does not repeat all tests for the second
# occurrence of AC_PROG_F77. So the first occurrence is the one that
# must always be performed, the second is optional. PG & CC sep.2006

if test "$f90" = "" ; then f90="$try_f90" ; fi
AC_PROG_F77($f90)
f90=$F77

if test "$use_parallel" -eq 0 ; 
then
	mpif90=$f90
else
        # clear cached values (not sure when and why this is needed)
        unset F77 ac_cv_prog_ac_ct_F77 ac_cv_f77_compiler_gnu ac_cv_prog_f77_g

	# check parallel Fortran 90 compiler
	if test "$mpif90" = "" ; then mpif90="$try_mpif90 $f90" ; fi
	AC_PROG_F77($mpif90)
	mpif90=$F77

	case "$arch" in
	ia32 | ia64 | amd64 )
		# check which compiler does mpif90 wrap
		echo $ECHO_N "checking version of $mpif90... $ECHO_C"
		ifort_version=`$mpif90 -V 2>&1 | grep "Intel(R)"`
		pgf_version=`$mpif90 -V 2>&1 | grep "^pgf"`
		eko_version=`$mpif90 -version 2>&1 | grep "EKOPath"`
		g95_version=`$mpif90 -v 2>&1 | grep "g95"`
		if test "$ifort_version" != ""
		then
			version=`$mpif90 -V 2>&1 | grep Version |
				 sed 's/.*Version//' | awk '{print $1}'`
			if test `echo $version | sed 's/\..*//'` -lt 8
			then
				try_f90="ifc efc"
				echo "${ECHO_T}ifc $version"
			else
				try_f90="ifort efc"
				echo "${ECHO_T}ifort $version"
			fi
		elif test "$pgf_version" != ""
		then
			version=`echo $pgf_version | awk '{print $2}'`
			try_f90="pgf90"
			echo "${ECHO_T}pgf90 $version"
		elif test "$eko_version" != ""
		then
			version=`echo $eko_version | awk '{print $6}'`
			try_f90="pathf95"
			echo "${ECHO_T}pathf95 $version"
		elif test "$g95_version" != ""
		then
			version=`echo $g95_version | awk '{print $3}'`
			try_f90="g95"
			echo "${ECHO_T}g95 $version"
		else
			echo "${ECHO_T}unknown"
		fi
		;;
	esac

fi

echo setting F90... $f90
echo setting MPIF90... $mpif90

# candidate C and f77 compilers good for all cases
try_cc="cc gcc"
try_f77="$f90"

case "$arch:$f90" in
*:ifort* )
	try_cc="icc ecc $try_cc"
	;;
*:ifc* )
	try_cc="icc ecc $try_cc"
	;;
*:pgf90 )
	try_cc="pgcc $try_cc"
	try_f77="pgf77 $f90"
	;;
*:pathf95 )
	try_cc="pathcc $try_cc"
	;;
ia32:* | ia64:* | amd64:* )
	try_f77="f77 $f90"
	;;
aix:*xlf*_r )
	try_cc="xlc_r $try_cc"
	try_f77="xlf_r f77 $f90"
	;;
aix:*xlf* )
	try_cc="xlc $try_cc"
	try_f77="xlf f77 $f90"
	;;
mips:* )
	try_f77="f77 $f90"
	;;
alpha:* )
	try_f77="f77 $f90"
	;;
alinux:* )
	try_cc="ccc $try_cc"
	try_f77="fort g77 f77 $f90"
	;;
sparc:* )
	try_f77="f77 $f90"
	;;
crayx1:ftn )
	try_cc="cc"
	;;
macppc:xlf90 )
	try_f77="xlf f77 $f90"
	;;
esac

# check serial C compiler
if test "$env_cc" = "" ; then cc="$try_cc" ; else cc="$env_cc"; fi
AC_PROG_CC($cc)
cc=$CC

echo setting CC... $cc

# check size of pointers to int - needed to decide the size of integer
# arrays in fortran holding C pointers for FFTW

AC_CHECK_SIZEOF([int *])
SIZEOF_INT_P=$ac_cv_sizeof_int_p
AC_SUBST(SIZEOF_INT_P)

# check if the structure mallinfo is present in malloc.h
AC_CHECK_HEADER(malloc.h,have_malloc_h=1,have_malloc_h=0, )
if test "$have_malloc_h" -ne 0 
then
AC_CHECK_MEMBER([struct mallinfo.arena], 
                [AC_DEFINE(HAVE_MALLINFO)], 
                ,
	        [#include <malloc.h>])

fi

# clear cached values
unset F77 ac_cv_prog_ac_ct_F77 ac_cv_f77_compiler_gnu ac_cv_prog_f77_g

# check serial Fortran 77 compiler
if test "$f77" = "" ; then f77="$try_f77" ; fi
AC_PROG_F77($f77)
f77=$F77

echo setting F77... $f77

# check Fortran compiler flags
have_cpp=1
xlf_flags=0

case "$arch:$f90" in
ia64:ifort* | amd64:ifort* )
        try_fautodbl="-r8"
	try_fflags="-O2 -assume byterecl"
	try_f90flags="\$(FFLAGS) -nomodule"
	try_fflags_noopt="-O0 -assume byterecl"
	try_ldflags=""
	try_ldflags_static="-static"
	try_dflags="$try_dflags -D__INTEL"
	pre_fdflags="-fpp "
	;;
ia64:ifc* )
        try_fautodbl="-r8"
	try_fflags="-Vaxlib -O2"
	try_f90flags="\$(FFLAGS) -nomodule"
	try_fflags_noopt="-O0"
	try_ldflags="-Vaxlib"
	try_ldflags_static="-static"
	try_dflags="$try_dflags -D__INTEL"
	pre_fdflags="-fpp "
	;;
ia64:pgf* | amd64:pgf* )
        try_fautodbl="-r8"
	try_fflags="-fast"
	try_f90flags="\$(FFLAGS)"
	try_fflags_noopt="-O0"
	try_ldflags=""
	try_ldflags_static="-Bstatic"
	try_dflags="$try_dflags -D__PGI"
	have_cpp=0
	;;
ia64:path* | amd64:path* )
        try_fautodbl="-r8"
	try_fflags="-march=auto -O2"
	try_f90flags="\$(FFLAGS)"
	try_fflags_noopt="-O0"
	try_ldflags=""
	try_ldflags_static="-static"
	try_dflags="$try_dflags -D__EKO"
	have_cpp=0
	;;
ia32:ifort* )
        try_fautodbl="-r8"
        try_fflags="-O2 -tpp6 -assume byterecl"
	try_f90flags="\$(FFLAGS) -nomodule"
	try_fflags_noopt="-O0 -assume byterecl"
	try_ldflags=""
	try_ldflags_static="-static"
	try_dflags="$try_dflags -D__INTEL"
	pre_fdflags="-fpp "
	;;
ia32:ifc* )
        try_fautodbl="-r8"
        try_fflags="-Vaxlib -O2 -tpp6"
	try_f90flags="\$(FFLAGS) -nomodule"
	try_fflags_noopt="-O0"
	try_ldflags="-Vaxlib"
	try_ldflags_static="-static"
	try_dflags="$try_dflags -D__INTEL"
	pre_fdflags="-fpp "

	if test `echo $version | sed 's/\..*//'` -lt 7
	then
		# old versions of ifc require this stuff
		try_f90flags="\$(FFLAGS) -cl,./intel.pcl"
                if test -x $topdir/ifcmods.sh
                then
                   ./ifcmods.sh
                fi
	fi
	;;
ia32:pgf* )
        try_fautodbl="-r8"
	try_fflags="-fast"
	try_f90flags="\$(FFLAGS)"
	try_fflags_noopt="-O0"
	try_ldflags=""
	try_ldflags_static="-Bstatic"
	try_dflags="$try_dflags -D__PGI"
	have_cpp=0
	;;
ia32:path* )
        try_fautodbl="-r8"
	try_fflags="-march=auto -O2"
	try_f90flags="\$(FFLAGS)"
	try_fflags_noopt="-O0"
	try_ldflags=""
	try_ldflags_static="-static"
	try_dflags="$try_dflags -D__EKO"
	have_cpp=0
	;;
*:g95 )
        try_fautodbl="-r8"
	try_fflags="-O3 -cpp"
	try_f90flags="\$(FFLAGS)"
	try_fflags_noopt="-O0 -cpp"
	try_ldflags=""
        try_dflags="$try_dflags -D__G95"
	#try_ldflags_static="-static"
	;;
*:gfortran )
        try_fautodbl="-r8"
	try_fflags="-O3"
	try_f90flags="\$(FFLAGS) -x f95-cpp-input"
	try_fflags_noopt="-O0"
	try_ldflags=""
	#try_ldflags_static="-static"
        ;;
aix:*xlf* )
        try_fautodbl="-qrealsize=8"
	try_fflags="-q64 -qalias=noaryovrlp -O3 -qstrict \
-qarch=auto -qtune=auto -qdpc -Q -qalias=nointptr"
	try_f90flags="\$(FFLAGS) -qsuffix=cpp=f90 -qfree=f90"
	try_fflags_noopt="-q64 -O0"
	try_ldflags="-q64"
	# try_ldflags_static="-bstatic"
	pre_fdflags="-WF,"
	xlf_flags=1
	;;
mips:f90 )
        try_fautodbl="-r8"
	try_fflags="-mips4 -64 -O2 -r10000 -r8"
	try_f90flags="\$(FFLAGS)"
	try_fflags_noopt="-mips4 -64 -O0"
	try_ldflags="-mips4 -64"
	pre_fdflags="-ftpp -macro_expand "
	;;
alinux:fort )
        try_fautodbl="-r8"
	pre_fdflags="-cpp "
	try_fflags="-O -align dcommons -align records"
	try_f90flags="\$(FFLAGS) -free"
	try_fflags_noopt="-O0 -align dcommons -align records"
        try_ldflags_static="-non_shared"
	;;
alpha:f90 )
        try_fautodbl="-real_size 64"
	pre_fdflags="-cpp "
	try_fflags="-O -align dcommons -align records -assume buffered_io -fpe1"
	try_f90flags="\$(FFLAGS) -free"
	try_fflags_noopt="-O0 -align dcommons -align records -assume buffered_io"
	;;
sparc:mpf90 | sparc:f90 )
        try_fautodbl=""
	try_fflags="-fast -O1 -nodepend -xvector=no -xchip=ultra3 \
-xarch=v8plusb -xlic_lib=sunperf"
	try_f90flags="\$(FFLAGS)"
	try_fflags_noopt="-O0 -xlic_lib=sunperf"
	try_ldflags=""
	imod="-M"
        have_cpp=0
	;;
crayx1:ftn )
        try_fautodbl=""
        try_fflags="-s default64 -dp -rma -e0 \
-O scalar3,stream3,vector3,nointerchange -Z -O inline5"
        try_f90flags="\$(FFLAGS)"
        try_fflags_noopt="-s default64 -dp -rma -e0 -O0 -Z"
        try_ldflags="-s default64 -f nan64"
        try_dflags="$try_dflags -D__X1_COA"
        pre_fdflags="-e Z -F"
	;;
macppc:* )
        try_fautodbl="-qrealsize=8"
	try_fflags="-O4 -qarch=auto -qtune=auto -qsuffix=cpp=f90 \
-qdpc -qalias=nointptr"
	try_f90flags="\$(FFLAGS) -qfree=f90"
	try_fflags_noopt="-O0"
	try_ldflags=""
	pre_fdflags="-WF,"
	xlf_flags=1
	;;
* )
	# unknown, try these
        try_fautodbl="-r8"
	try_fflags="-O"
	try_f90flags="\$(FFLAGS)"
	try_fflags_noopt="-O0"
	try_ldflags=""
	have_cpp=0
	;;
esac
if test "$use_shared" -eq 0 ; then
try_ldflags="$try_ldflags $try_ldflags_static" ; fi

if test "$fautodbl" = "" ; then fautodbl=$try_fautodbl ; fi
if test "$fflags" = ""   ; then fflags=$try_fflags     ; fi
if test "$f90flags" = "" ; then f90flags=$try_f90flags ; fi
if test "$fflags_noopt" = ""   ; then fflags_noopt=$try_fflags_noopt     ; fi
echo setting FFLAGS... $fflags
echo setting FFLAGS_NOOPT... $fflags_noopt
echo setting F90FLAGS... $f90flags
echo setting FAUTODBL... $fautodbl
if test "$imod" = "" ; then imod="-I" ; fi

# tentative C and loader flags, good for many cases
try_cflags="-O3"
c_ldflags=""
try_cpp="cpp"

case "$arch:$cc" in
*:pgcc )
	try_cflags="-fast"
	;;
*:pathcc )
	try_cpp="pathcc -E"
	;;
aix:mpcc* | aix:xlc* | aix:cc )
	try_cflags="-q64 -O2"
	c_ldflags="-q64"
	;;
mips:cc )
	try_cflags="-mips4 -64 -O2 -r10000"
	;;
alpha:cc )
	try_cflags="-O"
	;;
sparc:mpcc | sparc:cc )
	try_cflags="-fast -dalign -xchip=ultra3 -xarch=v8plusb \
-xlic_lib=sunperf"
	try_cpp="fpp"
	;;
crayx1:cc )
        try_cflags=""
        ;;
macppc:xlc* | macppc:mpcc* )
	try_cflags="-O4"
	;;
esac
if test "$cflags" = "" ; then cflags=$try_cflags ; fi
echo setting CFLAGS... $cflags

# preprocessor - try cpp in all cases; the preprocessor returned by
# AC_PROG_CPP
# may sometimes refuse to preprocess fortran files
if test "$cpp" = "" ; then cpp=$try_cpp; fi
# if test "$cpp" = "" ; then cpp=$CPP;     fi
echo setting CPP... $cpp

echo $ECHO_N "setting CPPFLAGS... $ECHO_C"
case $cpp in
	cpp)  try_cppflags="-P -traditional" ;;
	fpp)  try_cppflags="-P"              ;;
	*)    try_cppflags=""                ;;
esac
if test "$cppflags" = "" ; then cppflags=$try_cppflags ; fi
echo "${ECHO_T}$cppflags"

# linker and archiver
if test "$ld" = ""       ; then ld="$mpif90"           ; fi
if test "$ldflags" = ""  ; then ldflags="$try_ldflags" ; fi
if test "$ar" = ""       ; then ar="$try_ar"           ; fi
if test "$arflags" = ""  ; then arflags="$try_arflags" ; fi
if test "$arflags_dynamic" = ""  ; then arflags_dynamic="$try_arflags_dynamic"; fi
echo setting LD... $ld
echo setting LDFLAGS... $ldflags
echo setting AR... $ar
echo setting ARFLAGS... $arflags
echo setting ARFLAGS_DYNAMIC...$arflags_dynamic

# compilation rules
AC_PROG_MAKE_SET
echo $ECHO_N "checking whether Fortran files must be preprocessed... $ECHO_C"
if test "$have_cpp" -ne 0
then
	f90rule="\$(MPIF90) \$(F90FLAGS) -c \$<"
	echo "${ECHO_T}no"
else
	f90rule="\$(CPP) \$(CPPFLAGS) \$< -o \$*.F90 \\
	\$(MPIF90) \$(F90FLAGS) -c \$*.F90 -o \$*.o"
	echo "${ECHO_T}yes"
fi

# compilation flags for all subsequent tests
# remove all $(...) because at least one compiler doesn't like them
# but if f90flags contains $(FFLAGS), substitute it
test_cflags="`echo $cflags | sed 's/\$([[^)]]*)//g'`"
test_cppflags="$test_cflags"
if test "`echo $f90flags | grep '$(FFLAGS)'`" != ""
then
	test_fflags="`echo $fflags $f90flags | sed 's/\$([[^)]]*)//g'`"
else
	test_fflags="`echo $f90flags | sed 's/\$([[^)]]*)//g'`"
fi
test_ldflags="`echo $ldflags | sed 's/\$([[^)]]*)//g'`"

# find Fortran to C wrappers

AC_F77_WRAPPERS

AC_LANG_PUSH(Fortran 77)
F77=$f90 # use Fortran 90 actually 

# search for libraries
have_blas=0
have_atlas=0
have_mkl=0
have_lapack=0
have_essl=0
have_fft=0
have_fftw=0
have_fftw_h=0
have_mpi=0
#
blas_libs_switch=external
lapack_libs_switch=external
fft_libs_switch=external
#
internal_libs=

# check for blas
# supported vendor replacements:
#   mkl on linux
#   essl on aix
#   complib.sgimath on mips
#   cxml on alpha
#   SUNperf on sparc
# atlas is used over blas if available
# internal version is used if none is found
if test "$blas_libs" = ""
then
	# check directories in LD_LIBRARY_PATH too
	# (maybe they are already searched by default, but I'm not sure)
	ld_library_path=`echo $LD_LIBRARY_PATH | sed 's/:/ /g'`

	# build list of directories to search
	case "$arch" in
	macppc )    try_libdirs="/usr/local/lib /sw/lib" ;;
	*)       try_libdirs="/usr/local/lib"         ;;
	esac

	# search
	case "$arch:$f90" in

	amd64:pgf* )
		# check for acml
		unset ac_cv_search_dgemm # clear cached value
		FFLAGS="$test_fflags"
		LDFLAGS="$test_ldflags"
		LIBS=""
		AC_SEARCH_LIBS(dgemm, acml, have_blas=1 blas_libs="$LIBS")
		;;

	*:pgf* )
		# check for PGI blas
		unset ac_cv_search_dgemm # clear cached value
		FFLAGS="$test_fflags"
		LDFLAGS="$test_ldflags"
		LIBS=""
		AC_SEARCH_LIBS(dgemm, blas, have_blas=1 blas_libs="$LIBS")
		;;

	amd64:path* )
		# check for acml
                try_libdirs="/opt/acml*/pathscale64/lib/"
                try_libdirs="$libdirs $try_libdirs $ld_library_path"

		for dir in none $try_libdirs
		do
			unset ac_cv_search_dgemm # clear cached value
			if test "$dir" = "none"
			then
				try_loption=
			else
				echo $ECHO_N "in $dir: " $ECHO_C
				try_loption="-L$dir"
			fi
			FFLAGS="$test_fflags"
			LDFLAGS="$test_ldflags $try_loption"
 			LIBS=""
			AC_SEARCH_LIBS(dgemm, acml, have_blas=1 blas_libs="$try_loption $LIBS")
			if test "$ac_cv_search_dgemm" != "no"
			then break ; fi
		done
       	        ;;

	ia64:* )
		# check for mkl (in several directories)
		try_libdirs="/opt/intel/mkl/*/lib/64
			     /opt/intel/mkl*/lib/64"
		try_libdirs="$libdirs $try_libdirs $ld_library_path"

		for dir in none $try_libdirs
		do
			unset ac_cv_search_dgemm # clear cached value
			if test "$dir" = "none"
			then
				try_loption=
			else
				echo $ECHO_N "in $dir: " $ECHO_C
				try_loption="-L$dir"
			fi
			FFLAGS="$test_fflags"
			LDFLAGS="$test_ldflags $try_loption"
			LIBS="-lguide -lpthread"
			AC_SEARCH_LIBS(dgemm, mkl_ipf, have_blas=1 have_mkl=1
				       blas_libs="$try_loption $LIBS")
			if test "$ac_cv_search_dgemm" != "no"
			then break ; fi
		done
		;;

	amd64:* )
		try_libdirs="/opt/intel/mkl/*/lib/em64t
			     /opt/intel/mkl*/lib/em64t"
		try_libdirs="$libdirs $try_libdirs $ld_library_path"

		for dir in none $try_libdirs
		do
			unset ac_cv_search_dgemm # clear cached value
			if test "$dir" = "none"
			then
				try_loption=
			else
				echo $ECHO_N "in $dir: " $ECHO_C
				try_loption="-L$dir"
			fi
			FFLAGS="$test_fflags"
			LDFLAGS="$test_ldflags $try_loption"
			LIBS="-lguide -lpthread"
			AC_SEARCH_LIBS(dgemm, mkl_em64t, have_blas=1 have_mkl=1
				       blas_libs="$try_loption $LIBS")
			if test "$ac_cv_search_dgemm" != "no"
			then break ; fi
		done
		;;

	ia32:* )
		# check for mkl (in several directories)
		try_libdirs="/opt/intel/mkl/*/lib/32
			     /opt/intel/mkl*/lib/32
			     /cineca/prod/intel/lib"
		try_libdirs="$libdirs $try_libdirs $ld_library_path"

		for dir in none $try_libdirs
		do
			unset ac_cv_search_dgemm # clear cached value
			if test "$dir" = "none"
			then
				try_loption=
			else
				echo $ECHO_N "in $dir: " $ECHO_C
				try_loption="-L$dir"
			fi
			FFLAGS="$test_fflags"
			LDFLAGS="$test_ldflags $try_loption"
			LIBS="-lguide -lpthread"
			AC_SEARCH_LIBS(dgemm, mkl_ia32, have_blas=1 have_mkl=1
				       blas_libs="$try_loption $LIBS")
			if test "$ac_cv_search_dgemm" != "no"
			then break ; fi
		done
		;;

	aix:* )
		# check for essl
		unset ac_cv_search_dgemm # clear cached value
		FFLAGS="$test_fflags"
		LDFLAGS="$test_ldflags"
		LIBS=""
		AC_SEARCH_LIBS(dgemm, essl, have_blas=1
			       blas_libs="$LIBS" )
                # notice that some IBM machines may not need -lessl
                # to load blas so the above test may fail
                if test "`echo $blas_libs | grep essl`" != ""
                then
                    have_essl=1 
                    try_dflags="$try_dflags -D__ESSL"
                fi
		;;

	mips:* )
		# check for complib.sgimath
		unset ac_cv_search_dgemm # clear cached value
		FFLAGS="$test_fflags"
		LDFLAGS="$test_ldflags"
		LIBS=""
		AC_SEARCH_LIBS(dgemm, complib.sgimath, have_blas=1
			       blas_libs="$LIBS")
		;;

	alpha:* | alinux:* )
		# check for cxml
		unset ac_cv_search_dgemm # clear cached value
		FFLAGS="$test_fflags"
		LDFLAGS="$test_ldflags"
		LIBS=""
		AC_SEARCH_LIBS(dgemm, cxml, have_blas=1 have_lapack=1
                                blas_libs="$LIBS")
		;;

	sparc:* )
		# check for SUNperf library
		unset ac_cv_search_dgemm # clear cached value
		FFLAGS="$test_fflags"
		LDFLAGS="$test_ldflags"
		LIBS=""
		AC_SEARCH_LIBS(dgemm, sunperf, have_blas=1
			       blas_libs="-xlic_lib=sunperf $LIBS")
		;;

	esac

	if test "$have_blas" -eq 0
	then
		# check for atlas (in several directories)
		case "$f90" in
		ifort* | ifc* )
			try_libdirs="/usr/local/lib /cineca/prod/intel/lib" ;;
		pgf* )	try_libdirs="/usr/local/lib /cineca/prod/pgi/lib"   ;;
		g95 )	try_libdirs="/usr/local/lib /cineca/prod/gnu/lib"   ;;
		* )	try_libdirs="/usr/local/lib"                        ;;
		esac
		try_libdirs="$libdirs $try_libdirs $ld_library_path"

		for dir in none $try_libdirs
		do
			unset ac_cv_search_dgemm # clear cached value
			if test "$dir" = "none"
			then
				try_loption=
			else
				echo $ECHO_N "in $dir: " $ECHO_C
				try_loption="-L$dir"
			fi
			FFLAGS="$test_fflags"
			LDFLAGS="$test_ldflags $try_loption" 
			LIBS="-latlas" 
			AC_SEARCH_LIBS(dgemm, f77blas, have_blas=1 have_atlas=1
				       blas_libs="$try_loption $LIBS", , -lg2c)
			if test "$ac_cv_search_dgemm" != "no"
			then break ; fi
		done
	fi

	if test "$have_blas" -eq 0
	then
		# check for blas (in several directories)
		case "$f90" in
		ifort* | ifc* )
			try_libdirs="/usr/local/lib /cineca/prod/intel/lib" ;;
		pgf* )	try_libdirs="/usr/local/lib /cineca/prod/pgi/lib"   ;;
		g95 )	try_libdirs="/usr/local/lib /cineca/prod/gnu/lib"   ;;
		* )	try_libdirs="/usr/local/lib"                        ;;
		esac
		try_libdirs="$libdirs $try_libdirs $ld_library_path"

		for dir in none $try_libdirs
		do
			unset ac_cv_search_dgemm # clear cached value
			if test "$dir" = "none"
			then
				try_loption=
			else
				echo $ECHO_N "in $dir: " $ECHO_C
				try_loption="-L$dir"
			fi
			FFLAGS="$test_fflags"
			LDFLAGS="$test_ldflags $try_loption"
			LIBS=""
			AC_SEARCH_LIBS(dgemm, blas, have_blas=1
				       blas_libs="$try_loption $LIBS")
			if test "$ac_cv_search_dgemm" != "no"
			then break ; fi
		done
	fi
fi

# check for lapack
# supported vendor replacements:
#   essl on IBM
#   mkl on linux
#   SUNperf on sparc
# internal version is used if none is found
if test "$lapack_libs" = ""
then
	# check directories in LD_LIBRARY_PATH too
	# (maybe they are already searched by default, but I'm not sure)
	ld_library_path=`echo $LD_LIBRARY_PATH | sed 's/:/ /g'`

	# build list of directories to search
	case "$arch" in
	macppc )    try_libdirs="/usr/local/lib /sw/lib" ;;
	*)       try_libdirs="/usr/local/lib"         ;;
	esac

	# search
	case "$arch:$f90" in

	ia32:* | ia64:* )
		# check for mkl_lapack
		if test "$have_mkl" -ne 0
		then
			unset ac_cv_search_zggev # clear cached value
			FFLAGS="$test_fflags"
			LDFLAGS="$test_ldflags"
			LIBS="$blas_libs"
			AC_SEARCH_LIBS(zggev, mkl_lapack, have_lapack=1
				       lapack_libs="-lmkl_lapack")
		fi
		;;

        amd64:path* )
                # check for acml
                unset ac_cv_search_zggev # clear cached value
                FFLAGS="$test_fflags"
                LDFLAGS="$test_ldflags"
                LIBS="$blas_libs"
                AC_SEARCH_LIBS(zggev, acml, have_lapack=1 lapack_libs="$LIBS")
                ;;

	amd64:* )
		# check for mkl_lapack
		if test "$have_mkl" -ne 0
		then
			unset ac_cv_search_zggev # clear cached value
			FFLAGS="$test_fflags"
			LDFLAGS="$test_ldflags"
			LIBS="$blas_libs"
			AC_SEARCH_LIBS(zggev, mkl_lapack, have_lapack=1
				       lapack_libs="-lmkl_lapack")
		fi
		;;

	sparc:* )
		# check for SUNperf library
		unset ac_cv_search_zggev # clear cached value
		FFLAGS="$test_fflags"
		LDFLAGS="$test_ldflags"
		LIBS="$blas_libs"
		AC_SEARCH_LIBS(zggev, sunperf, have_lapack=1
			       lapack_libs="-xlic_lib=sunperf $LIBS")
		;;
	aix:* )
		# check for essl (may not be revealed by the previous
		# test of blas) 
                if test have_essl -eq 0
                then
		   unset ac_cv_search_zggev # clear cached value
		   FFLAGS="$test_fflags"
		   LDFLAGS="$test_ldflags"
		   LIBS="$blas_libs"
		   AC_SEARCH_LIBS(dgemm, essl, have_essl=1
		  	          lapack_libs="-lessl" 
 				  try_dflags="$try_dflags -D__ESSL")
                fi
		;;
	esac

	if test "$have_lapack" -eq 0
	then
		# check for lapack (in several directories)
		case "$f90" in
		ifort* | ifc* )
			try_libdirs="/usr/local/lib /cineca/prod/intel/lib" ;;
		pgf* )	try_libdirs="/usr/local/lib /cineca/prod/pgi/lib"   ;;
		g95 )	try_libdirs="/usr/local/lib /cineca/prod/gnu/lib"   ;;
		*xlf* ) try_libdirs="/usr/local/lib /cineca/lib"            ;;
		* )	try_libdirs="/usr/local/lib"                        ;;
		esac
		try_libdirs="$libdirs $try_libdirs $ld_library_path"

		for dir in none $try_libdirs
		do
			unset ac_cv_search_zggev # clear cached value
			if test "$dir" = "none"
			then
				try_loption=
			else
				echo $ECHO_N "in $dir: " $ECHO_C
				try_loption="-L$dir"
			fi
			FFLAGS="$test_fflags"
			LDFLAGS="$test_ldflags $try_loption"
			LIBS="$blas_libs"
			AC_SEARCH_LIBS(zggev, lapack, have_lapack=1
				       lapack_libs="$try_loption -llapack")
			if test "$ac_cv_search_zggev" != "no"
			then break ; fi
		done
	fi

	# essl must precede lapack (if present)

	if test "$have_essl" -ne 0 && test "$have_lapack" -ne 0
	then lapack_libs="-lessl $lapack_libs" ; fi

        # if some or all lapack routines are missing, add them here

        if test "$have_lapack" -eq 0 ; then

            if test "$have_atlas" -ne 0 ; then

            # add missing lapack routines so as to complete atlas
            # note that some compilers do not like to have multiple symbols

            #
            # Andrea Ferretti & Layla Martin Samos did it
            #
            # Since we do no t have local copies of the missing atlas routines,
            # set the use the full internal lapack only (atlas only for blas)
            #
	    #  lapack_libs="$lapack_libs ../libs/lapack_atlas.a"
            #  internal_libs="$internal_libs lapack_atlas.a"
            #
 	    lapack_libs="$topdir/lapack/src/liblapack.a"
            lapack_libs_switch="internal"
            internal_libs="$internal_libs lapack.a"

        elif test "$have_essl" -ne 0 ; then

        # ibm doesn't complain if there are multiple symbols so add everything
 
            #
            # Andrea Ferretti & Layla Martin Samos did it
            #
            # internal lapck should be compiled to complete missing essl
            # in our case, no need for this at the moment, so do not waste time
            # and forget about internal lapack. To be fixed whenever the case.
            #
            #  lapack_libs="$lapack_libs $topdir/lapack/src/liblapack.a"
            #  lapack_libs_switch="internal"
            #  internal_libs="$internal_libs lapack.a"
            #
            lapack_libs="$lapack_libs $topdir/lapack/src/liblapack.a"
            lapack_libs_switch="internal"
            internal_libs="$internal_libs lapack.a"

        else

	# nothing found, use built-in library

 	    lapack_libs="$topdir/lapack/src/liblapack.a"
            lapack_libs_switch="internal"
            internal_libs="$internal_libs lapack.a"

        fi
    fi
fi

# no blas library found: use the built-in blas
# (blas_libs is used in the above tests so it is safe to set it here)

if test "$have_blas" -eq 0 -a "$blas_libs" = ""
then
    blas_libs="$topdir/blas/src/libblas.a"
    blas_libs_switch="internal"
    internal_libs="$internal_libs blas.a"
fi

echo setting BLAS_LIBS... $blas_libs
echo setting BLAS_LIBS_SWITCH... $lapack_libs_switch
echo setting LAPACK_LIBS... $lapack_libs
echo setting LAPACK_LIBS_SWITCH... $lapack_libs_switch


# check for FFT libraries
# supported vendor replacements:
#   essl on aix
#   SUNperf on sparc
# internal version is used if none is found
if test "$fft_libs" = ""
then
	# check directories in LD_LIBRARY_PATH too
	# (maybe they are already searched by default, but I'm not sure)
	ld_library_path=`echo $LD_LIBRARY_PATH | sed 's/:/ /g'`

	# build list of directories to search
	case "$arch" in
	macppc )    try_libdirs="/usr/local/lib /sw/lib" ;;
	*)       try_libdirs="/usr/local/lib"         ;;
	esac

	# search
	case "$arch" in
	aix )
		# check for essl
		unset ac_cv_search_dcft # clear cached value
		FFLAGS="$test_fflags"
		LDFLAGS="$test_ldflags"
		LIBS="$fft_libs"
		AC_SEARCH_LIBS(dcft, essl, have_fft=1 fft_libs="$LIBS")
	    ;;     
	sparc )
		# check for SUNperf library
		unset ac_cv_search_zfft3i # clear cached value
		FFLAGS="$test_fflags"
		LDFLAGS="$test_ldflags"
		LIBS="$libs"
		AC_SEARCH_LIBS(zfft3i, sunperf, have_fft=1
			       fft_libs="-xlic_lib=sunperf $LIBS")
		;;
	esac

	if test "$have_fft" -eq 0
	then
		AC_LANG_PUSH(C)

		# check for fftw (in several directories)
		case "$f90" in
		ifort* | ifc* )
			try_libdirs="/usr/local/lib /cineca/prod/intel/lib" ;;
		pgf* )	try_libdirs="/usr/local/lib /cineca/prod/pgi/lib"   ;;
		* )	try_libdirs="/usr/local/lib /cineca/prod/gnu/lib"   ;;
		esac
		try_libdirs="$libdirs $try_libdirs $ld_library_path"

		for dir in none $try_libdirs
		do
			unset ac_cv_search_fftwnd # clear cached value
			if test "$dir" = "none"
			then
				try_loption=
			else
				echo $ECHO_N "in $dir: " $ECHO_C
				try_loption="-L$dir"
			fi
			CFLAGS="$test_cflags"
			CPPFLAGS="$test_cppflags"
			LDFLAGS="$c_ldflags $try_loption"
			LIBS="$fft_libs"
			AC_SEARCH_LIBS(fftwnd, fftw, have_fftw=1
				       fft_libs="$try_loption $LIBS", , -lm)
			if test "$ac_cv_search_fftwnd" != "no"
			then break ; fi
		done

		AC_LANG_POP(C)
	fi
fi

# final checks on available FFT libraries

if test "$have_fft" -eq 0
then
	AC_LANG_PUSH(C)

	try_dflags="$try_dflags -D__FFTW"

	CFLAGS="$test_cflags"
	LDFLAGS="$c_ldflags"
	LIBS="$fft_libs"
	AC_SEARCH_LIBS(fftwnd, "", have_fft=1 have_fftw=1)

	# find location of fftw.h (if needed)
	if test "$have_fftw" -ne 0
	then
		if test "$includefftw" = ""
		then
			# if libfftw is in /some/path/lib, then fftw.h is
			# probably in /some/path/include
			includefftw=`echo $fft_libs |
				sed 's/.*-L\([[^ ]]*\)  *-lfftw.*/\1/
				     s/lib/include/'`
		fi
		try_includedirs="$includefftw /usr/local/include \
				 /cineca/lib/fftw-2.1.3/fftw"
                for dir in none $try_includedirs
		do
			unset ac_cv_header_fftw_h # clear cached value
			if test "$dir" = "none"
			then
				try_ifftw=
			else
				echo $ECHO_N "in $dir: " $ECHO_C
				try_ifftw="-I$dir"
			fi
			CFLAGS="$test_cflags $try_ifftw"
			CPPFLAGS="$test_cppflags $try_ifftw"
			LDFLAGS="$c_ldflags"
			LIBS="$fft_libs"
			AC_CHECK_HEADER(fftw.h, have_fftw_h=1
					try_iflags="$try_iflags $try_ifftw")
			if test "$ac_cv_header_fftw_h" != "no"
			then break ; fi
		done
	fi

	# if no valid FFT library was found, use the local copy
	if test "$have_fft" -eq 0
	then
		try_dflags="$try_dflags -D__USE_INTERNAL_FFTW"
	fi

	if test "$have_fftw" -ne 0 && test "$have_fftw_h" -eq 0
	then
		try_dflags="$try_dflags -D__USE_INTERNAL_FFTW"
		AC_MSG_WARN([fftw library detected, but fftw.h not found])
	fi

	AC_LANG_POP(C)
fi

#
# get the switch for FFT
#
if test "$have_fft" -eq 0
then
        fft_libs_switch="internal"
        fft_libs="$topdir/fftw/src/libfftw.a"
        fft_libs_include="-I$topdir/fftw/include"
fi
echo setting FFT_LIBS_INCLUDE... $fft_libs_include
echo setting FFT_LIBS_SWITCH... $fft_libs_switch
echo setting FFT_LIBS... $fft_libs


F77=$f90
FFLAGS="$test_fflags"
LDFLAGS="$test_ldflags"
# check for mass on aix
if test "$mass_libs" = ""
then
	# check directories in LD_LIBRARY_PATH too
	# (maybe they are already searched by default, but I'm not sure)
	ld_library_path=`echo $LD_LIBRARY_PATH | sed 's/:/ /g'`

	# build list of directories to search
	case "$arch" in
	macppc )    try_libdirs="/usr/local/lib /sw/lib" ;;
	*)       try_libdirs="/usr/local/lib"         ;;
	esac

	# search
	case "$arch" in
	aix )
		# check for mass (in several directories)
		try_libdirs="/usr/local/lib /cineca/lib /cineca/lib/mass"
		try_libdirs="$libdirs $try_libdirs $ld_library_path"

		for dir in none $try_libdirs
		do
			unset ac_cv_search_vexp # clear cached value
			if test "$dir" = "none"
			then
				try_loption=
			else
				echo $ECHO_N "in $dir: " $ECHO_C
				try_loption="-L$dir"
			fi
			FFLAGS="$test_fflags"
			LDFLAGS="$test_ldflags $try_loption"
			LIBS=""
			AC_SEARCH_LIBS(vexp, massvp4 massv, , , -lmass)
			if test "$ac_cv_search_vexp" = "-lmassvp4" \
				-o "$ac_cv_search_vexp" = "-lmassv"
			then mass_libs="$try_loption $ac_cv_search_vexp -lmass"
			fi
			if test "$ac_cv_search_vexp" != "no" ; then break ; fi
		done
		;;
	esac
fi

if test "$mass_libs" != ""; then try_dflags="$try_dflags -D__MASS" ; fi

echo setting MASS_LIBS... $mass_libs

# check for mpi
# some architectures require to link mpi libraries explicitly
F77=$mpif90 # use parallel compiler
if test "$mpi_libs" = ""
then
	# check directories in LD_LIBRARY_PATH too
	# (maybe they are already searched by default, but I'm not sure)
	ld_library_path=`echo $LD_LIBRARY_PATH | sed 's/:/ /g'`

	# build list of directories to search
	case "$arch" in
	macppc )    try_libdirs="/usr/local/lib /sw/lib" ;;
	*)       try_libdirs="/usr/local/lib"         ;;
	esac

	# search
	if test "$use_parallel" -ne 0
	then
		case "$arch" in
		alpha )
			AC_LANG_PUSH(C)
			CFLAGS="$test_cflags"
			CPPFLAGS="$test_cppflags"
			LDFLAGS="$c_ldflags"
			LIBS=""
			AC_SEARCH_LIBS(elan_baseInit, elan, mpi_libs="$LIBS")
			LIBS="-lmpi $mpi_libs"
			AC_SEARCH_LIBS(mpi_waitany_, fmpi, have_mpi=1
				       mpi_libs="$LIBS")
			AC_LANG_POP(C)

			LIBS="$mpi_libs"
			AC_SEARCH_LIBS(pmpi_init, pmpi, mpi_libs="$LIBS")
			;;
		esac

		if test "$have_mpi" -eq 0
		then
			# check for mpi
			unset ac_cv_search_mpi_init # clear cached value
			LDFLAGS="$test_ldflags"
			LIBS="$mpi_libs"
			AC_SEARCH_LIBS(mpi_init, mpi,
				       have_mpi=1 mpi_libs="$LIBS")
		fi
	fi
fi
echo setting MPI_LIBS... $mpi_libs

# final check on availability of parallel environment
parallel=0
if test "$use_parallel" -ne 0
then
	F77=$mpif90
	LIBS="$mpi_libs"
	AC_SEARCH_LIBS(mpi_init, "", parallel=1
		       try_dflags="$try_dflags -D__MPI -D__PARA")
fi

if test "$dflags" = "" ; then dflags="$try_dflags" ; fi
echo setting DFLAGS... $dflags

if test "$iflags" = "" ; then iflags="$try_iflags" ; fi
echo setting IFLAGS... $iflags

# xlf compilers (AIX and powerpc) want comma-separated -D directives
if test "$xlf_flags" -ne 0
then
	fdflags="`echo $dflags | sed 's/  */,/g'`"
else
	fdflags="\$(DFLAGS)"
fi
echo setting FDFLAGS... $fdflags

AC_CHECK_PROG(ranlib,ranlib,ranlib,echo)
echo setting RANLIB... $ranlib

# configure output messages
blas_line="BLAS_LIBS=$blas_libs"
lapack_line="LAPACK_LIBS=$lapack_libs"
fft_line="FFT_LIBS=$fft_libs"
if test "$have_fft" -ne 0 && test "$have_fftw" -ne 0 &&
	test "$have_fftw_h" -eq 0
then
	fft_line="$fft_line FFT_LIBS=$fft_libs\\
  WARNING: fftw library detected, but fftw.h not found"
fi
if test "$mpi_libs" != "" ; then mpi_line="MPI_LIBS=$mpi_libs"
else mpi_line="@delete@" ; fi
if test "$arch" = "aix" ; then mass_line="MASS_LIBS=$mass_libs"
else mass_line="@delete@" ; fi

if test "$use_parallel" -ne 0
then
	if test "$parallel" -ne 0
	then
		parallel_report="Parallel environment detected successfully.\\
Configured for compilation of parallel executables."
	else
		parallel_report="Parallel environment not detected \
\(is this a parallel machine?\).\\
Configured for compilation of serial executables."
	fi
else
	parallel_report="Configured for compilation of serial executables."
fi

# export settings to generated files
AC_SUBST(host)
AC_SUBST(arch)
AC_SUBST(cc)
AC_SUBST(cflags)
AC_SUBST(dflags)
AC_SUBST(fdflags)
AC_SUBST(cpp)
AC_SUBST(cppflags)
AC_SUBST(f90)
AC_SUBST(mpif90)
AC_SUBST(f90flags)
AC_SUBST(f77)
AC_SUBST(fflags)
AC_SUBST(fautodbl)
AC_SUBST(fflags_noopt)
AC_SUBST(pre_fdflags)
AC_SUBST(imod)
AC_SUBST(iflags)
AC_SUBST(includefftw)
AC_SUBST(ld)
AC_SUBST(ldflags)
AC_SUBST(blas_libs)
AC_SUBST(lapack_libs)
AC_SUBST(fft_libs)
AC_SUBST(mpi_libs)
AC_SUBST(mass_libs)
AC_SUBST(pgplot_libs)
#
AC_SUBST(blas_libs_switch)
AC_SUBST(lapack_libs_switch)
AC_SUBST(fft_libs_switch)
AC_SUBST(fft_libs_include)
AC_SUBST(internal_libs)
#
AC_SUBST(ar)
AC_SUBST(arflags)
AC_SUBST(arflags_dynamic)
AC_SUBST(ranlib)
AC_SUBST(f90rule)
AC_SUBST(blas_line)
AC_SUBST(lapack_line)
AC_SUBST(fft_line)
AC_SUBST(mpi_line)
AC_SUBST(mass_line)
AC_SUBST(parallel_report)

#
# tmp removed
# AC_CONFIG_FILES([include/configure.h:include/configure.h.in])
#

if test -d $topdir/ffti
then 
    # 
    AC_CONFIG_FILES([$topdir/ffti/include/ffti_fdefs.h:$topdir/ffti/include/ffti_fdefs.h.in])
    AC_CONFIG_HEADER([$topdir/ffti/include/ffti_cdefs.h:$topdir/ffti/include/ffti_cdefs.h.in])
    #
fi

if test -d $topdir/ctools
then 
    # 
    AC_CONFIG_HEADER([$topdir/ctools/include/ctools.h:$topdir/ctools/include/ctools.h.in])
    #
fi

#
AC_CONFIG_FILES(make.sys)
AC_CONFIG_FILES(configure.msg)

AC_OUTPUT

#----------------------
# configure iotk
#
# generate dependencies
if test -x $topdir/iotk/tools/configure
then
        echo
        echo configuring iotk...
        #
        if test "$iotk_integer_kinds" = "" ; then iotk_integer_kinds=ALL ; fi
        if test "$iotk_logical_kinds" = "" ; then iotk_logical_kinds=ALL ; fi
        if test "$iotk_real_kinds" = ""    ; then iotk_real_kinds=ALL    ; fi
        if test "$iotk_fc" = ""            ; then iotk_fc=$f90           ; fi
        echo setting IOTK_INTEGER_KINDS... $iotk_integer_kinds
        echo setting IOTK_LOGICAL_KINDS... $iotk_logical_kinds
        echo setting IOTK_REAL_KINDS... $iotk_real_kinds
        echo setting IOTK_FC... $iotk_fc
        echo
        #
        ( cd $topdir ; ./iotk/tools/configure IOTK_INTEGER_KINDS=$iotk_integer_kinds \
                                              IOTK_LOGICAL_KINDS=$iotk_logical_kinds \
                                              IOTK_REAL_KINDS=$iotk_real_kinds \
                                              FC=$iotk_fc )
fi

#----------------------
# final messages
echo
sed '/@delete@/d' configure.msg

echo configure: success
