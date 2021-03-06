SHELL = /bin/bash

##
# Definitions.

.SUFFIXES:

PACKAGE_NAME = {{ cookiecutter.repo_name }}
PACKAGE_VERSION = {{ cookiecutter.version }}

srcdir = $(CURDIR)/$(PACKAGE_NAME)
builddir = $(CURDIR)/build
distdir = $(CURDIR)/dist
{%- if cookiecutter.enable_sphinx == 'true' %}
docdir = $(CURDIR)/doc
autodocdir = $(docdir)/autodoc
{%- endif %}

## OS specifics.

ifneq ($(shell which lsb_release),)
	OS_ID = $(shell lsb_release -is)
	OS_CODENAME = $(shell lsb_release -cs)
endif

## Tools.

tools =

ifeq ($(shell uname -s),Darwin)
	SED = gsed
else
	SED = sed
endif

ifeq ($(shell which ${SED}),)
	tools += $(SED)
endif

GREPTOOL = ack
ifeq ($(shell which ${GREPTOOL}),)
    GREPTOOL = egrep
endif

AWK = awk
ifeq ($(shell which ${AWK}),)
    tools += $(AWK)
endif

DOCKER = docker
ifeq ($(shell which ${DOCKER}),)
    tools += $(DOCKER)
endif

{%- if cookiecutter.enable_vagrant == 'true' %}

VAGRANT = vagrant
ifeq ($(shell which ${VAGRANT}),)
	tools += $(VAGRANT)
endif
{%- endif %}

## Virtual environment.

VENV_DIR = $(CURDIR)/.venv

VENV = virtualenv
ifeq ($(shell which ${VENV}),)
	tools += $(VENV)
endif

## Docker.

PYTHON_VERSION = {{ cookiecutter.python_version }}
DOCKER_BASE_IMAGE = python:$(PYTHON_VERSION)

DOCKER_PS_ARGS = -s
DOCKER_WORK_DIR = /var/app

DOCKER_WORK_DIR = /usr/local/lib/python$(PYTHON_VERSION)/site-packages/$(PACKAGE_NAME)

DOCKER_PORT = 8088

{%- if cookiecutter.enable_sphinx == 'true' %}

## Documentation.

doctools =

SPHINX_BUILDDIR = $(docdir)/_build
SPHINX_STATIC = $(docdir)/_static
SPHINX_TEMPLATES = $(docdir)/_templates

SPHINX_OPTS = -d $(SPHINX_BUILDDIR)/doctrees $(CURDIR)/doc

SPHINX = sphinx-build
ifeq ($(shell which ${SPHINX}),)
	doctools += $(SPHINX)
endif

SPHINX_APIDOC = sphinx-apidoc
ifeq ($(shell which ${SPHINX_APIDOC}),)
	doctools += $(SPHINX_APIDOC)
endif

AUTODOC_EXCLUDE_MODULES =

PDFLATEX = pdflatex
ifeq ($(shell which ${PDFLATEX}),)
	doctools += $(PDFLATEX)
endif
{%- endif %}


##
# All.

all: help
ifdef tools
	$(error "Can't find tools:${tools}")
endif


##
# Virtual environment.

.PHONY: requirements
# target: requirements – Compile Pip requirements
requirements:
	@if [[ ! -f requirements.txt ]]; then \
		touch requirements.txt; \
	fi
	@$(DOCKER) run -it --rm \
		-v "$(CURDIR)/requirements.in:/requirements.in" \
		-v "$(CURDIR)/requirements.txt:/requirements.txt" \
	"$(DOCKER_BASE_IMAGE)" "$(SHELL)" -c \
		'pip install pip-tools && CUSTOM_COMPILE_COMMAND="make requirements" \
		pip-compile -o requirements.tmp /requirements.in && \
		cat requirements.tmp > requirements.txt'

$(VENV_DIR): requirements.txt requirements-test.txt
	@$(VENV) -p {{ cookiecutter.python }} "$(VENV_DIR)"
	@"$(VENV_DIR)/bin"/pip install -U setuptools pip
	@"$(VENV_DIR)/bin"/pip install -Ur $<
	@"$(VENV_DIR)/bin"/pip install -Ur $(word 2,$^)

.PHONY: venv
# target: venv – Create the virtual environment
venv: $(VENV_DIR)


##
# Docker.

.PHONY: docker-info
# target: docker-info - Display system-wide information
docker-info:
	@echo
	@$(DOCKER) info
	@echo

.PHONY: docker-stats
# target: docker-stats - Show all images and containers
docker-stats:
	@echo
	@$(DOCKER) images -a
	@echo
	@$(DOCKER) ps -a
	@echo

.PHONY: docker-statsall
# target: docker-statsall - Same as `stats`, but more details provided
docker-statsall:
	@echo
	@$(DOCKER) images -a
	@echo
	@$(DOCKER) ps -a $(DOCKER_PS_ARGS)
	@echo

.PHONY: docker-build
# target: docker-build - Build image from scratch
docker-build: distclean dist
	@$(DOCKER) build \
		-f "$(CURDIR)/Dockerfile" -t "$(PACKAGE_NAME):$(PACKAGE_VERSION)" \
		--no-cache .

.PHONY: docker-run
# target: docker-run - Run temporary container in an interactive mode
docker-run:
	@$(DOCKER) run -it --rm -p "$(DOCKER_PORT):$(DOCKER_PORT)" \
		"$(PACKAGE_NAME):$(PACKAGE_VERSION)"

.PHONY: docker-clean
# target: docker-clean – Clean dangling images
docker-clean:
	@$(DOCKER) rmi -f \
		$(shell $(DOCKER) images -a | $(GREPTOOL) "<none>" | $(AWK) '{print $$3}') \
	&>/dev/null || :

.PHONY: docker-distclean
# target: docker-distclean – Clean built containers
docker-distclean:
	@$(DOCKER) rm -f $(shell $(DOCKER) ps -aq) &>/dev/null || :

.PHONY: docker-mostlyclean
# target: docker-mostlyclean – Remove all unused images, built containers and volumes
docker-mostlyclean: docker-distclean
	@$(DOCKER) image prune -a
	@$(DOCKER) volume prune -f


##
# Building and packaging.

.PHONY: dist
# target: dist – Create a binary (wheel) distribution
dist:
	@[ ! -f "$(distdir)"/*.whl ]
	@python setup.py bdist_wheel

.PHONY: sdist
# target: sdist – Create a source distribution
sdist:
	@[ ! -f "$(distdir)"/*.tar.gz ]
	@python setup.py sdist

.PHONY: install
# target: install – Install project sources in "development mode"
install:
	@python setup.py develop

.PHONY: uninstall
# target: uninstall – Uninstall project sources
uninstall:
	@python setup.py develop --uninstall


##
# Testing.

.PHONY: check
# target: check – Run tests
check:
	@python setup.py test -a "-vv"

{%- if cookiecutter.enable_sphinx == 'true' %}


##
# Documentation.

.PHONY: doc
doc:
ifdef doctools
	$(error "Can't find tools:${doctools}")
endif

.PHONY: apidoc
# target: apidoc - Create one reST file with automodule directives per package
apidoc: doc
	@$(SPHINX_APIDOC) --force --private -o "$(autodocdir)" $(PACKAGE_NAME) \
		$(foreach module,$(AUTODOC_EXCLUDE_MODULES),$(PACKAGE_NAME)/$(module))

.PHONY: html
# target: html – Render standalone HTML files
html: doc
	@$(SPHINX) -b html $(SPHINX_OPTS) "$(SPHINX_BUILDDIR)/html"

.PHONY: pdf
# target: pdf – Generate LaTeX files and run them through pdflatex
pdf: doc
	@$(SPHINX) -b latex $(SPHINX_OPTS) "$(SPHINX_BUILDDIR)/latex" && \
		$(MAKE) -C "$(SPHINX_BUILDDIR)/latex" all-pdf
{%- endif %}

{%- if cookiecutter.enable_vagrant == 'true' %}


##
# Vagrant.

.PHONY: vagrant-update
# target: vagrant-update - Update the machine within current Vagrant environment
vagrant-update:
	@$(VAGRANT) box update

.PHONY: vagrant-up
# target: vagrant-up – Start and provision the Vagrant environment
vagrant-up:
	@$(VAGRANT) up

.PHONY: vagrant-ssh
# target: vagrant-ssh – Connect to the machine via SSH as root
vagrant-ssh:
	@$(VAGRANT) ssh -- -l root

.PHONY: vagrant-halt
# target: vagrant-halt – Stop the machine
vagrant-halt:
	@$(VAGRANT) halt

.PHONY: vagrant-destroy
# target: vagrant-destroy – Stop and delete all traces of the machine
vagrant-destroy:
	@$(VAGRANT) destroy
{%- endif %}


##
# Auxiliary targets.

.PHONY: help
# target: help – Display all callable targets
help:
	@echo
	@egrep "^\s*#\s*target\s*:\s*" [Mm]akefile \
	| $(SED) -r "s/^\s*#\s*target\s*:\s*//g"
	@echo

## Cleaners.

.PHONY: clean
# target: clean – Clean the project's directrory
clean:
	@find "$(CURDIR)" \
		-name "*.py[cod]" -exec rm -fv {} + -o \
		-name __pycache__ -exec rm -rfv {} +
	@rm -rfv \
		"$(CURDIR)/.cache" \
		"$(CURDIR)/.mypy_cache" \
		"$(CURDIR)/.pytest_cache"

.PHONY: distclean
# target: distclean – Clean the project's build output
distclean:
	@find "$(CURDIR)" -path "$(VENV_DIR)" -prune -o \
		-name "*.egg-info" -exec rm -rfv {} + -o \
		-name "*.dist-info" -exec rm -rfv {} +

	@rm -rfv \
		"$(CURDIR)/.eggs" \
		"$(builddir)" \
		"$(distdir)" \
{%- if cookiecutter.enable_sphinx == 'true' %}
		"$(SPHINX_BUILDDIR)" \
		"$(SPHINX_STATIC)" \
		"$(SPHINX_TEMPLATES)"
	@rm -fv "$(docdir)/autodoc"/*.rst
{%- endif %}

.PHONY: mostlyclean
# target: mostlyclean – Delete almost everything
mostlyclean: clean distclean
	@find "$(CURDIR)" -name .DS_Store -exec rm -fv {} +

	@rm -rfv \
{%- if cookiecutter.enable_vagrant == 'true' %}
		"$(CURDIR)/.vagrant" \
{%- endif %}
		"$(VENV_DIR)"
