.PHONY: install check playbook

install:
	@echo "Installing dependencies..."
	pip install ansible

check:
	@echo "Checking Ansible version..."
	ansible --version

	@echo "Setting up ansible.cfg..."
	echo '[defaults]\nroles_path=../' > ansible.cfg

check_syntax:
	@echo "Running syntax check..."
	ansible-playbook tests/test.yml -i tests/inventory --syntax-check
	
playbook:
	@echo "Running syntax check..."
	ansible-playbook tests/test.yml -i tests/inventory 

notify:
	@echo "Sending notification..."
	curl -X POST https://galaxy.ansible.com/api/v1/notifications/

all: install check playbook
