# +------------------------------------------------------------------+
# |             ____ _               _        __  __ _  __           |
# |            / ___| |__   ___  ___| | __   |  \/  | |/ /           |
# |           | |   | '_ \ / _ \/ __| |/ /   | |\/| | ' /            |
# |           | |___| | | |  __/ (__|   <    | |  | | . \            |
# |            \____|_| |_|\___|\___|_|\_\___|_|  |_|_|\_\           |
# |                                                                  |
# | Copyright Mathias Kettner 2014             mk@mathias-kettner.de |
# +------------------------------------------------------------------+
#
# This file is part of Check_MK.
# The official homepage is at http://mathias-kettner.de/check_mk.
#
# check_mk is free software;  you can redistribute it and/or modify it
# under the  terms of the  GNU General Public License  as published by
# the Free Software Foundation in version 2.  check_mk is  distributed
# in the hope that it will be useful, but WITHOUT ANY WARRANTY;  with-
# out even the implied warranty of  MERCHANTABILITY  or  FITNESS FOR A
# PARTICULAR PURPOSE. See the  GNU General Public License for more de-
# ails.  You should have  received  a copy of the  GNU  General Public
# License along with GNU Make; see the file  COPYING.  If  not,  write
# to the Free Software Foundation, Inc., 51 Franklin St,  Fifth Floor,
# Boston, MA 02110-1301 USA.


SHELL         = /bin/bash
VERSION       = 1.2.7i4
VERSION_DUMMY = @@@@@CHECK_MK_VERSION@@@@@
WINDRES       = i686-w64-mingw32-windres
#COPTIONS      = -s -O2 -Wall -Wformat=2 -Werror -static -static-libgcc -static-libstdc++ -fno-rtti -std=c++11 -DCHECK_MK_VERSION='"$(VERSION_DUMMY)"' -D__USE_MINGW_ANSI_STDIO
# for debug builds
COPTIONS      = -O0 -Wall -Werror -static -static-libgcc -static-libstdc++ -fno-rtti -std=c++11 -ggdb -DCHECK_MK_VERSION='"$(VERSION_DUMMY)"' -D__USE_MINGW_ANSI_STDIO
MKDIR         = mkdir -p

ifeq ($(shell uname),Linux)
  CXX        = i686-w64-mingw32-g++
  CXX_64     = x86_64-w64-mingw32-g++
  WINDRES_64 = x86_64-w64-mingw32-windres
else
  CXX = mingw32-g++
endif

all: directories fix_linebreaks check_mk_agent.exe check_mk_agent-64.exe install_agent.exe install_agent-64.exe msi_installer

.PHONY: directories inc_build_version fix_linebreaks

setversion:
	sed -i 's/!define CHECK_MK_VERSION .*/!define CHECK_MK_VERSION "'$(NEW_VERSION)'"/' installer*.nsi ; \
	sed -ri 's/^(VERSION[[:space:]]*:?= *).*/\1'"$(NEW_VERSION)/" Makefile ; \
	rm check_mk_agent.exe check_mk_agent-64.exe check_mk_agent.msi ; \
	make install_agent.exe install_agent-64.exe msi_installer

# Converts all files which are targeted to a windows system to CRLF
fix_linebreaks:
	for F in mrpe/* plugins/* *.ini; do \
	    if [ -f "$$F" ]; then \
	        sed -i 's/\r//' "$$F" ; \
	        sed -i 's/$$/\r/' "$$F" ; \
	    fi ; \
	done

# -----------------------------------------------------------------------------
# 1. Create ressource files with version and product information. Note: That file
# contains a dummy version number, which is later being replaced with the
# actual version number, during 'make version'
# -----------------------------------------------------------------------------
inc_build_version:
	echo $$(( $$(cat build_version) + 1 )) > build_version.new && mv build_version.new build_version
	BUILD_VERSION=$$(cat build_version) ; \
	   sed -e "s/%VERSION%/$(VERSION_DUMMY)/g" \
	       -e "s/%YEAR%/$$(date +%Y)/g" \
	       -e "s/%BUILD_VERSION%/$$BUILD_VERSION/g" check_mk_agent.rc.in > check_mk_agent.rc

check_mk_agent.res: check_mk_agent.rc
	$(WINDRES) $< -O coff -o $@

check_mk_agent-64.res: check_mk_agent.rc
	$(WINDRES_64) $< -O coff -o $@

# -----------------------------------------------------------------------------
# 2. Create directories for the object files
# -----------------------------------------------------------------------------

directories:
	${MKDIR} obj
	${MKDIR} obj64


# -----------------------------------------------------------------------------
# 3. Compile the Windows agent from the sources, without a specific version
# number. Instead a placeholder is being used as version.a
# -----------------------------------------------------------------------------

obj/%.o: %.cc
	$(CXX) -c $(COPTIONS) -o $@  $<

obj64/%.o: %.cc
	$(CXX_64) -c $(COPTIONS) -o $@  $<


check_mk_agent.unversioned.exe: obj/check_mk_agent.o obj/SettingsCollector.o obj/Configuration.o obj/Environment.o obj/stringutil.o obj/wmiHelper.o obj/ListenSocket.o obj/PerfCounter.o
	$(MAKE) inc_build_version
	$(MAKE) check_mk_agent.res
	$(CXX) $(COPTIONS) -o $@ $^ check_mk_agent.res -lwsock32 -lole32 -loleaut32 -lwbemuuid -lpsapi

check_mk_agent-64.unversioned.exe: obj64/check_mk_agent.o obj64/SettingsCollector.o obj64/Configuration.o obj64/Environment.o obj64/stringutil.o obj64/wmiHelper.o obj64/ListenSocket.o obj64/PerfCounter.o
	$(MAKE) inc_build_version
	$(MAKE) check_mk_agent-64.res
	$(CXX_64) $(COPTIONS) -o $@ $^ check_mk_agent-64.res -lwsock32 -lole32 -loleaut32 -lwbemuuid -lpsapi


# -----------------------------------------------------------------------------
# 4. Create final executable files by inserting actual version number. No
# recompilation is thus needed during 'make version'
# -----------------------------------------------------------------------------

check_mk_agent.exe.versioned:
	./bin_replace $(VERSION_DUMMY) "$(VERSION)" < check_mk_agent.unversioned.exe > check_mk_agent.exe
	chmod 755 check_mk_agent.exe

check_mk_agent-64.exe.versioned:
	./bin_replace $(VERSION_DUMMY) "$(VERSION)" < check_mk_agent-64.unversioned.exe > check_mk_agent-64.exe
	chmod 755 check_mk_agent-64.exe


%.exe: %.unversioned.exe %.exe.versioned
	true

# -----------------------------------------------------------------------------
# NSIS installer will be removed and replaced by multiarch MSI installer
# -----------------------------------------------------------------------------
install_agent.exe: check_mk_agent.exe.versioned crash.exe nowin.exe
	if ! which makensis; then \
	    echo "ERROR: makensis missing. Please install the nsis package." ; \
	    exit 1 ; \
	fi
	makensis installer.nsi
	chmod +x install_agent.exe

install_agent-64.exe: check_mk_agent-64.exe.versioned crash.exe nowin.exe
	if ! which makensis; then \
	    echo "ERROR: makensis missing. Please install the nsis package." ; \
	    exit 1 ; \
	fi
	makensis installer-64.nsi
	chmod +x install_agent-64.exe

msi_installer:
	cd msibuild ; make BUILD_VERSION=$$(cat ../build_version) CHECK_MK_VERSION=$(VERSION)
	mv msibuild/check_mk_agent.msi .
	chmod +x check_mk_agent.msi

crash.exe: crash.cc
	$(CXX) -o $@ $<

nowin.exe: nowin.cc
	$(CXX) -o $@ $<

clean:
	rm -f *.o *.exe *~ svn-*.tmp *.rc *.res obj/*.o obj64/*.o
	make -C msibuild clean

mrproper: clean
	rm -f *.exe

setup:
	sudo apt-get install nsis lcab binutils-mingw-w64-i686 binutils-mingw-w64-x86-64
