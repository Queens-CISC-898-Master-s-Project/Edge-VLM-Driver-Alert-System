# EdgeVLM-Drive Architecture

## Project Goal

Develop an offline driver distraction and drowsiness warning system using a lightweight Vision-Language Model running on NVIDIA Jetson Nano.

## Hardware

- NVIDIA Jetson Nano 4GB
- USB Camera
- Buzzer
- SD Card

## Software

- Ubuntu
- ROS 2
- Python
- Docker
- GitHub

## System Workflow

Camera
↓

Frame Sampling
↓

Vision-Language Model

↓

Driver State

↓

Temporal Decision Logic

↓

Buzzer Alert

## Driver States

- Normal
- Drowsy
- Looking Away
- Phone Usage

## Research Objectives

- Latency
- Accuracy
- Memory
- Power Consumption