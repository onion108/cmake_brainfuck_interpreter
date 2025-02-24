# vimtcompile:[ -d build ] && rm -rf build; cmake -B build
cmake_minimum_required(VERSION 3.31)
project(BrainfuckInterpreter)

set(INPUT_FILE test.bf CACHE FILEPATH "Input file path")
set(STDIN_FILE program_in CACHE FILEPATH "Path of standard input for the program")

# Read file into FILE_CONTENT.
file(READ "${INPUT_FILE}" FILE_CONTENT)
file(READ "${STDIN_FILE}" STDIN_CONTENT)
string(LENGTH "${STDIN_CONTENT}" STDIN_CONTENT_LEN)
set(STDIN_PTR 0)

set(STDOUT_BUF "")
function(putchar char)
    if(char STREQUAL "\n")
        message("${STDOUT_BUF}")
        set(STDOUT_BUF "" PARENT_SCOPE)
        return()
    endif()
    set(STDOUT_BUF "${STDOUT_BUF}${char}" PARENT_SCOPE)
endfunction()

function(getchar outvar)
    if(STDIN_PTR LESS STDIN_CONTENT_LEN)
        string(SUBSTRING "${STDIN_CONTENT}" ${STDIN_PTR} 1 ch)
        char2ascii(ch ${ch})
        set(${outvar} ${ch} PARENT_SCOPE)
        math(EXPR ret "${STDIN_PTR}+1")
        set(STDIN_PTR ${ret} PARENT_SCOPE)
        return()
    endif()
    set(${outvar} -1 PARENT_SCOPE)
endfunction()

function(flush)
    if(NOT STDOUT_BUF STREQUAL "")
        message("${STDOUT_BUF}")
        set(STDOUT_BUF "" PARENT_SCOPE)
    endif()
endfunction()

function(char2ascii outvar char)
    string(SUBSTRING "${char}" 0 1 ch)
    foreach(i RANGE 1 256)
        string(ASCII ${i} dest)
        if(ch STREQUAL dest)
            set(${outvar} ${i} PARENT_SCOPE)
            return()
        endif()
    endforeach()
    set(${outvar} 0 PARENT_SCOPE)
endfunction()

string(LENGTH "${FILE_CONTENT}" FILE_CONTENT_LEN)
set(PROGRAM_COUNTER 0)
set(POINTER 0)
set(RAM 0 0 0 0 0)
list(LENGTH RAM RAM_CAP)

while(${PROGRAM_COUNTER} LESS ${FILE_CONTENT_LEN})
    string(SUBSTRING "${FILE_CONTENT}" ${PROGRAM_COUNTER} 1 ch)
    if(ch STREQUAL "+")
        list(GET RAM ${POINTER} value)
        list(REMOVE_AT RAM ${POINTER})
        math(EXPR value "${value}+1")
        list(INSERT RAM ${POINTER} ${value})
    elseif(ch STREQUAL "-")
        list(GET RAM ${POINTER} value)
        list(REMOVE_AT RAM ${POINTER})
        math(EXPR value "${value}-1")
        list(INSERT RAM ${POINTER} ${value})
    elseif(ch STREQUAL "<")
        math(EXPR POINTER "${POINTER}-1")
        if(POINTER LESS 0)
            message(FATAL_ERROR "Cannot move pointer below address 0")
        endif()
    elseif(ch STREQUAL ">")
        math(EXPR POINTER "${POINTER}+1")
        while(RAM_CAP LESS_EQUAL ${POINTER})
            list(APPEND RAM 0)
            list(LENGTH RAM RAM_CAP)
        endwhile()
    elseif(ch STREQUAL "[")
        list(GET RAM ${POINTER} ret)
        if("${ret}" EQUAL 0)
            set(left_count 0)
            set(found_match OFF)
            while(PROGRAM_COUNTER LESS FILE_CONTENT_LEN)
                string(SUBSTRING "${FILE_CONTENT}" ${PROGRAM_COUNTER} 1 ch)
                if(ch STREQUAL "[")
                    math(EXPR left_count "${left_count}+1")
                elseif(ch STREQUAL "]")
                    # Compare to 1 because the initial bracket will cause left_count to increase
                    if(left_count EQUAL 1)
                        set(found_match ON)
                        break()
                    else()
                        math(EXPR left_count "${left_count}-1")
                    endif()
                endif()
                math(EXPR PROGRAM_COUNTER "${PROGRAM_COUNTER}+1")
            endwhile()
            if(NOT found_match)
                message(FATAL_ERROR "Unbalanced left bracket found in source")
            endif()
        endif()
    elseif(ch STREQUAL "]")
        list(GET RAM ${POINTER} ret)
        if(NOT "${ret}" EQUAL 0)
            set(right_count 0)
            set(found_match OFF)
            while(PROGRAM_COUNTER GREATER_EQUAL 0)
                string(SUBSTRING "${FILE_CONTENT}" ${PROGRAM_COUNTER} 1 ch)
                if(ch STREQUAL "]")
                    math(EXPR right_count "${right_count}+1")
                elseif(ch STREQUAL "[")
                    # Compare to 1 because the initial bracket will cause left_count to increase
                    if(right_count EQUAL 1)
                        set(found_match ON)
                        break()
                    else()
                        math(EXPR right_count "${right_count}-1")
                    endif()
                endif()
                math(EXPR PROGRAM_COUNTER "${PROGRAM_COUNTER}-1")
            endwhile()
            if(NOT found_match)
                message(FATAL_ERROR "Unbalanced right bracket found in source")
            endif()
        endif()
    elseif(ch STREQUAL ".")
        list(GET RAM ${POINTER} ret)
        string(ASCII ${ret} ret)
        putchar(${ret})
    elseif(ch STREQUAL ",")
        getchar(ret)
        list(REMOVE_AT RAM ${POINTER})
        list(INSERT RAM ${POINTER} ${ret})
    endif()
    math(EXPR PROGRAM_COUNTER "${PROGRAM_COUNTER}+1")
endwhile()

flush()

