## Ruby 설치 및 Jekyll 실행 가이드

이 가이드는 macOS 환경에서 `rbenv`를 이용한 Ruby 설치부터 `Jekyll` 서버 실행까지의 과정을 단계별로 설명합니다.

---
## rbenv install
brew install rbenv

## ruby install
rbenv install 3.2.2   # 원하는 버전으로 변경 가능
rbenv global 3.2.2

## adjust
echo 'eval "$(rbenv init -)"' >> ~/dev/env
source ~/dev/env

## Bundler install
gem install bundler -v 2.6.8
bundle install
bundle exec jekyll serve