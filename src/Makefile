# Copyright © 2022 Kris Nóva <nova@nivenly.org>
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

CC       = clang

default: compile ## Default to compile

.PHONY: clean
clean: ## Clean objects
	rm -vf upstream-*
	rm -vf *.o
	rm -vf *.ll

compile:  ## Compile local code
	@echo "  ->  Compile upstream"
	${CC} basic-server-tcp.c -o upstream-basic-server-tcp
	${CC} basic-server-unix.c -o upstream-basic-server-unix
	${CC} server-not-accept-unix.c -o upstream-server-not-accept-unix
	${CC} server-accept-not-read-unix.c -o upstream-server-accept-not-read-unix
	${CC} server-accept-read-not-write-unix.c -o upstream-server-accept-read-not-write-unix
	${CC} server-accept-read-write-not-close-unix.c -o upstream-server-accept-read-write-not-close-unix
	@echo "  ->  Compile okay!"

.PHONY: help
help:  ## Show help messages for make targets
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(firstword $(MAKEFILE_LIST)) | sort | awk 'BEGIN {FS = ":.*?## "}; {//printf "\033[32m%-30s\033[0m %s\n", $$1, $$2}'
