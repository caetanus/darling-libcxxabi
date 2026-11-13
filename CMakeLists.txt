project(libcxxabi)

# DARLING: libcxxabi stays at llvmorg-13.0.1 (the libcxx-15 upgrade kept this
# subproject pinned because LLVM 15's libcxxabi expects libcxx-15's internal
# headers (`include/atomic_support.h`, std::unexpected_handler typedefs, etc.)
# in arrangements that diverge from Darling's vendoring layout. Pairing libc++15
# (consumer-facing) with libc++abi 13 (ABI side) is safe — Apple itself ships
# matched/mismatched versions and the ABI is stable across these versions.
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -nostdinc -std=c++11")
set(CMAKE_SHARED_LINKER_FLAGS "-nostdlib")

include_directories(BEFORE
	${CMAKE_CURRENT_SOURCE_DIR}/../libcxx/include
	${CMAKE_CURRENT_SOURCE_DIR}/../libcxx
	# libcxx's internal helpers (atomic_support.h etc.); libcxxabi 13 sources
	# include them via "include/atomic_support.h" — needs libcxx/src on the path.
	${CMAKE_CURRENT_SOURCE_DIR}/../libcxx/src
)

# OSXER (aligned allocation): the C++17 aligned operator new/delete were compiled out, although
# lib/new-delete.exp exports them and macOS 10.13+ has them; C++ libraries built for current
# macOS (conda-forge's libc++ re-exports this libc++abi) failed to load without them.
add_definitions(-D__DARWIN_NON_CANCELABLE=1
	# DARLING: libcxxabi 13 source needs symbols that libcxx 15 removed/gated:
	# - _LIBCPP_SAFE_STATIC was a require_constant_initialization attribute;
	#   gone in libcxx 15. Empty define is safe (semantic is best-effort).
	# - std::unexpected_handler removed in C++17 unless this opt-in macro set.
	-D_LIBCPP_SAFE_STATIC=
	-D_LIBCPP_ENABLE_CXX17_REMOVED_UNEXPECTED_FUNCTIONS=1
	-D_LIBCPP_BUILDING_LIBRARY=1
)

set(cxxabi_sources
	src/abort_message.cpp
	src/cxa_aux_runtime.cpp
	src/cxa_default_handlers.cpp
	src/cxa_demangle.cpp
	src/cxa_exception.cpp
	src/cxa_exception_storage.cpp
	src/cxa_guard.cpp
	src/cxa_handlers.cpp
	#src/cxa_noexception.cpp
	src/cxa_personality.cpp
	src/cxa_thread_atexit.cpp
	src/cxa_vector.cpp
	src/cxa_virtual.cpp
	src/fallback_malloc.cpp
	src/private_typeinfo.cpp
	src/stdlib_exception.cpp
	src/stdlib_new_delete.cpp
	src/stdlib_stdexcept.cpp
	src/stdlib_typeinfo.cpp
)

add_darling_object_library(cxxabi_obj ${cxxabi_sources})
make_fat(cxxabi_obj)

set(DYLIB_INSTALL_NAME "/usr/lib/libc++abi.dylib")
set(DYLIB_COMPAT_VERSION "1.0.0")
set(DYLIB_CURRENT_VERSION "1.0.0")

# OSXER (aligned allocation): the static copy links into dyld, which has no posix_memalign;
# it keeps the aligned operators compiled out. The dylib below gets them.
add_darling_object_library(cxxabi_obj_static ${cxxabi_sources})
target_compile_definitions(cxxabi_obj_static PRIVATE _LIBCPP_HAS_NO_LIBRARY_ALIGNED_ALLOCATION=1)
make_fat(cxxabi_obj_static)
add_darling_static_library(cxxabi_static SOURCES $<TARGET_OBJECTS:cxxabi_obj_static>)
make_fat(cxxabi_static)

add_darling_library(cxxabi SHARED $<TARGET_OBJECTS:cxxabi_obj>)
target_link_libraries(cxxabi system)
set_target_properties(cxxabi PROPERTIES OUTPUT_NAME "c++abi")
#set_property(TARGET cxxabi APPEND_STRING PROPERTY LINK_FLAGS " -Wl,-undefined,dynamic_lookup") # Needed for weakly imported symbols
make_fat(cxxabi)

install(TARGETS cxxabi DESTINATION libexec/darling/usr/lib)

