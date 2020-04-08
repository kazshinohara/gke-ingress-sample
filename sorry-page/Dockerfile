FROM golang:1.12-alpine
COPY . /go/src/sorry-page
RUN go install sorry-page

FROM alpine:latest
COPY --from=0 /go/bin/sorry-page .
ENV PORT 8080
CMD ["./sorry-page"]
