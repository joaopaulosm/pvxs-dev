PYTHON?=python3
CURL?=curl

# documentation generation requires Doxygen and Sphinx with the breathe extension
#
# apt-get install doxygen python3-sphinx python3-breathe

all: gen

clean:
	rm -rf xml default pvxs.tag
	rm -rf html _build
	rm -rf epics-base.inv

fetch: epics-base.inv

# conf.py reads CONFIG_PVXS_VERSION to extract version number
gen: ../include/pvxs/version.h epics-base.inv
	doxygen
	$(PYTHON) -m sphinx -j auto -b html . html
	cp qsrv2-schema-0.json html/

commit: gen
	./commit-gh.sh documentation/html/ html/.nojekyll html/*.* html/_*/*.*

epics-base.inv:
	$(CURL) -o "$@" "https://docs.epics-controls.org/en/latest/objects.inv"

.PHONY: all clean fetch gen commit
