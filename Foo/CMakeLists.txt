# Этот файл создаёт проект 'Foo', который компилируется в две цели - библиотеки 'bar' и 'baz'.
# Цель 'bar' зависит от 'baz'. После установки этот проект может быть найден
# с помощью команды 'find_package(... CONFIG)':
#
#    find_package(Foo CONFIG REQUIRED)
#    target_link_libraries(... Foo::bar)
#
# Замечание: требования распространаются автоматически, например:
#   * Foo::baz линкуется автоматически
#   * <prefix>/include added to header search path
#   * FOO_BAZ_DEBUG=1/FOO_BAR_DEBUG=1 добавляется в конфигурации Debug
#   * FOO_BAZ_DEBUG=0/FOO_BAR_DEBUG=0 добавляется в других конфигурациях

####
# Установим минималльную версию CMake. Поскольку команда 'project' использует
# параметр VERSION, нам необходима версия 3.0.
cmake_minimum_required(VERSION 3.0)

####
# Зададим переменные:
#   * PROJECT_NAME
#   * PROJECT_VERSION
project(Foo VERSION 1.2.3)

####
# Создадим цели
add_library(bar "Source/foo/Bar.cpp" "Source/foo/Bar.hpp")
add_library(baz "Source/foo/Baz.cpp" "Source/foo/Baz.hpp")

####
# Свойства целей

# Добавим определения для целей
# Значения:
#   * Debug: -DFOO_BAR_DEBUG=1
#   * Release: -DFOO_BAR_DEBUG=0
#   * другие: -DFOO_BAR_DEBUG=0
target_compile_definitions(bar PUBLIC "FOO_BAR_DEBUG=$<CONFIG:Debug>")
target_compile_definitions(baz PUBLIC "FOO_BAZ_DEBUG=$<CONFIG:Debug>")

# Генерируем:
#   * ${CMAKE_CURRENT_BINARY_DIR}/generated_headers/foo/BAR_EXPORT.h с BAR_EXPORT
#   * ${CMAKE_CURRENT_BINARY_DIR}/generated_headers/foo/BAZ_EXPORT.h с BAZ_EXPORT
# Переименовываем по причине:
# * Нам требуется префикс 'foo' to fit OSX/iOS frameworks layout
# * Имя файла совпадает с именем макроса
set(generated_headers "${CMAKE_CURRENT_BINARY_DIR}/generated_headers")
set(bar_export "${generated_headers}/foo/BAR_EXPORT.h")
set(baz_export "${generated_headers}/foo/BAZ_EXPORT.h")
include(GenerateExportHeader)
generate_export_header(bar EXPORT_FILE_NAME ${bar_export})
generate_export_header(baz EXPORT_FILE_NAME ${baz_export})

# Глобальные включаемые файлы. Используются всеми целями
# Замечания:
#   * расположение заголовочного файла в проекте: Foo/Source/foo/Bar.hpp
#   * может быть включен с помощью строки: `#include <foo/Bar.hpp>`
#   * расположение заголовочного файла в проекте: ${CMAKE_CURRENT_BINARY_DIR}/generated_headers/foo/BAR_EXPORT.h
#   * может быть включен с помощью строки: `#include <BAR_EXPORT.h>`
target_include_directories(
    baz PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/Source>"
    "$<BUILD_INTERFACE:${generated_headers}>"
)

# Линкуем требующуюся библиотеку
target_link_libraries(bar PUBLIC baz)

# Установка (https://github.com/forexample/package-example) {

# Расположение на всех платформах:
#   * <prefix>/lib/cmake/<PROJECT-NAME>
#   * <prefix>/lib/
#   * <prefix>/include/
set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

# Конфигурация
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

# Включаем модуль с функцией 'write_basic_package_version_file'
include(CMakePackageConfigHelpers)

# Конфигурируем '<PROJECT-NAME>ConfigVersion.cmake'
# Используем:
#   * PROJECT_VERSION
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# Конфигурируем '<PROJECT-NAME>Config.cmake'
# Используются переменные:
#   * TARGETS_EXPORT_NAME
#   * PROJECT_NAME
configure_package_config_file(
    "cmake/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

# Цели:
#   * <prefix>/lib/libbar.a
#   * <prefix>/lib/libbaz.a
#   * после установки, заголовочный файл находится здесь: <prefix>/include/foo/Bar.hpp
#   * заголовочные файлы могут быть включены строкой: `#include <foo/Bar.hpp>`
install(
    TARGETS bar baz
    EXPORT "${TARGETS_EXPORT_NAME}"
    LIBRARY DESTINATION "lib"
    ARCHIVE DESTINATION "lib"
    RUNTIME DESTINATION "bin"
    INCLUDES DESTINATION "${include_install_dir}"
)

# Заголовочные файлы:
#   * Source/foo/Bar.hpp -> <prefix>/include/foo/Bar.hpp
#   * Source/foo/Baz.hpp -> <prefix>/include/foo/Baz.hpp
install(
    DIRECTORY "Source/foo"
    DESTINATION "${include_install_dir}"
    FILES_MATCHING PATTERN "*.hpp"
)

# Экспортируемые заголовочные файлы:
#   * ${CMAKE_CURRENT_BINARY_DIR}/.../BAR_EXPORT.h -> <prefix>/include/foo/BAR_EXPORT.h
#   * ${CMAKE_CURRENT_BINARY_DIR}/.../BAZ_EXPORT.h -> <prefix>/include/foo/BAZ_EXPORT.h
install(
    FILES "${bar_export}" "${baz_export}"
    DESTINATION "${include_install_dir}/foo"
)

# Config
#   * <prefix>/lib/cmake/Foo/FooConfig.cmake
#   * <prefix>/lib/cmake/Foo/FooConfigVersion.cmake
install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

# Config
#   * <prefix>/lib/cmake/Foo/FooTargets.cmake
install(
    EXPORT "${TARGETS_EXPORT_NAME}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)

# }
