# Introduction

<!--
  Copyright 2022 Cargill Incorporated
  Licensed under Creative Commons Attribution 4.0 International License
  https://creativecommons.org/licenses/by/4.0/
-->

Augrim is a collection of reusable consensus algorithms.

## Using Augrim's Algorithms

To use an algorithm, your application will commonly need to do two primary things:

- Input events into the `Algorithm::event` method.
- Process the list of actions returned by the `Algorithm::event` method.

The specific actions and events differ by algorithm; refer to the algorithm's documentation.
