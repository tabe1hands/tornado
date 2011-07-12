SPHINXOPTS=-W -d sphinx/build/doctrees sphinx

.PHONY: sphinx
sphinx:
	sphinx-build -b html $(SPHINXOPTS) sphinx/build/html

.PHONY: sphinx-ja
sphinx-ja:
	sphinx-build -b html $(SPHINXOPTS) sphinx-ja/build/html


.PHONY: coverage
coverage:
	sphinx-build -b coverage ${SPHINXOPTS} sphinx/build/coverage

clean:
	rm -rf sphinx/build
	rm -rf sphinx-ja/build
