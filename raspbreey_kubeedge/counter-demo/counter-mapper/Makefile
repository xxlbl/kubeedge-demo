.PHONY: all pi-execute-app docker clean

all: pi-execute-app

pi-execute-app:
	go build -o pi-counter-app main.go

docker:
	docker build . -t kubeedge/kubeedge-pi-counter:arm64

clean:
	rm -f pi-counter-app
