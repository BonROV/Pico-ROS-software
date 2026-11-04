# Pico-ROS barebones talker/listener for the Waveshare ESP32-P4-NANO (Ethernet).
#
#   idf.py set-target esp32p4
#   idf.py build flash monitor
cmake_minimum_required(VERSION 3.16)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(picoros_esp32p4_nano)
