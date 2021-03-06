if (${CMAKE_SYSTEM_NAME} STREQUAL "Windows")
    set(LLDB_LinkDylib "liblldb" CACHE STRING "" FORCE)
else()
    get_filename_component(LibPython ${Python3_LIBRARY_RELEASE} NAME_WLE)
    string(SUBSTRING ${LibPython} 3 -1 LibPython) # strip "lib" prefix
    set(LLDB_LinkDylib "lldb;${LibPython}" CACHE STRING "" FORCE)
endif()

get_filename_component(LibPythonDir ${Python3_LIBRARY_RELEASE} DIRECTORY)
set(LLDB_LinkSearch "${LLDB_ROOT}/lib;${LibPythonDir}" CACHE STRING "" FORCE)

set(CargoFlags ${CargoFlags} --manifest-path=${CMAKE_SOURCE_DIR}/Cargo.toml
                             --target=${LLVM_TRIPLE} --target-dir=${CMAKE_BINARY_DIR}/target
                             -Zpackage-features)
if (CMAKE_BUILD_TYPE MATCHES Release|RelWithDebInfo)
    set(CargoFlags ${CargoFlags} --release)
    set(CargoOutDir ${CMAKE_BINARY_DIR}/target/${LLVM_TRIPLE}/release)
else()
    set(CargoOutDir ${CMAKE_BINARY_DIR}/target/${LLVM_TRIPLE}/debug)
endif()

if (CMAKE_VERBOSE_MAKEFILE)
    set(CargoFlags ${CargoFlags} -vv)
endif()

add_copy_files_to(PythonFiles ${CMAKE_CURRENT_BINARY_DIR}
        ${CMAKE_SOURCE_DIR}/adapter/codelldb.py
        ${CMAKE_SOURCE_DIR}/adapter/value.py
        ${CMAKE_SOURCE_DIR}/adapter/debugger.py
        ${CMAKE_SOURCE_DIR}/formatters/rust.py
)

add_custom_target(codelldb ALL
    DEPENDS codelldb_bin codelldb_lib
)

add_custom_target(codelldb_bin
    COMMAND cargo build --package=codelldb --bin codelldb --features=weak-linkage ${CargoFlags}
    COMMAND ${UpdateFile} ${CargoOutDir}/codelldb${ExeSuffix} ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Building codelldb_bin"
    USES_TERMINAL
)

add_custom_target(codelldb_lib
    DEPENDS lldb ${PythonFiles}
    COMMAND cargo build --package=codelldb --lib --features=weak-linkage ${CargoFlags}
    COMMAND ${UpdateFile} ${CargoOutDir}/${DylibPrefix}codelldb${DylibSuffix} ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Building codelldb_lib"
    USES_TERMINAL
)

add_custom_target(codelldb_syms
    DEPENDS codelldb_bin codelldb_lib
    COMMAND ${UpdateFile} ${CargoOutDir}/deps/${DylibPrefix}codelldb${DylibSuffix}${SymSuffix} ${CMAKE_CURRENT_BINARY_DIR}
    COMMAND ${UpdateFile} ${CargoOutDir}/codelldb${ExeSuffix}${SymSuffix} ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Updating codelldb symbols"
    USES_TERMINAL
)

set(PATH $ENV{PATH})
get_filename_component(PythonRuntimeDir "${Python3_RUNTIME_LIBRARY_RELEASE}" DIRECTORY)
string(JOIN ${PathSep} PATH ${PATH} ${LLDB_ROOT}/bin ${PythonRuntimeDir})
string(JOIN ${PathSep} LD_LIBRARY_PATH $ENV{LD_LIBRARY_PATH} ${LLDB_ROOT}/lib)

add_custom_target(cargo_test
    COMMAND ${WithEnv} PATH=\"${PATH}\" LD_LIBRARY_PATH=\"${LD_LIBRARY_PATH}\" RUST_BACKTRACE=1
            cargo test --lib --target=${LLVM_TRIPLE} -p=lldb -p=debug-protocol -p=codelldb
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    USES_TERMINAL
)

# Create dependency check tests
foreach (Binary codelldb${ExeSuffix} ${DylibPrefix}codelldb${DylibSuffix})
    add_test(NAME dependencies:${Binary}
        COMMAND ${Python3_EXECUTABLE} ${CMAKE_SOURCE_DIR}/tools/check_dependencies.py ${CMAKE_CURRENT_BINARY_DIR}/${Binary} ${AllowedDependencies})
endforeach()
