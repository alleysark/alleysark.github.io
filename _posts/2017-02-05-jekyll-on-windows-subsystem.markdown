---
layout: post
title: "Windows subsystem에 jekyll 세팅"
date: 2017-02-05 23:23:03
author: "alleysark"
image: '/assets/images/'
tags: [windows bash, jekyll, github-page]
---

jekyll은 정적 사이트 생성기로 간단한 설정만으로 손쉽게 블로그를 생성할 수 있습니다.
본 블로그도 jekyll을 사용하여 생성되었고, github page로 서비스되고 있죠.

문제가 하나 있는데, Windows에서는 jekyll을 사용하기 굉-장히 까다로웠다는 점입니다. 설치도 힘들고 사용하기도 영 맘에 안들었습니다.

그런데 Windows 10에 subsystem으로 ubuntu bash shell을 사용할 수 있게되었습니다!

최근에 Windows 10을 구매해서 바로 bash를 설치하고 jekyll 세팅에 들어갔습니다.
그러면서 몇몇 이슈를 겪어 기록 및 공유 차원에서 글을 작성합니다.

## 요구사항
기본적으로 Windows 10 subsystem이 설정 및 설치되어 있어야 합니다.
설치한 적이 없다면 [윈도우 10에서 Bash shell 지원](https://blogs.msdn.microsoft.com/eva/?p=7633)글을 따라하시면 됩니다.

## Ruby 설치

### rvm 설치
jekyll은 ruby로 작성된 정적 사이트 생성기입니다. 따라서 ruby가 필요합니다. 기본적으로 bash에는 ruby가 설치되어 있지 않습니다. 그런데 apt-get으로 설치하려 하면 큰일납니다. jekyll이 필요로하는 ruby 최소버전은 2.x 이상입니다. 그런데 apt에 등록된 ruby stable 버전은 1.9입니다. 그러므로 apt-get install ruby 하면 굉장히 옛 버전의 ruby가 설치되며, 다시 지우기도 굉장히 까탈스럽습니다.

루비 버전은 굉장히 많고, 현재 jekyll을 깔기위한 버전 외에 다른 프로그램 개발을 위해 다른 버전의 ruby가 필요해질 수도 있으므로 ruby의 여러 버전을 관리해 줄 매니저 프로그램을 먼저 설치합니다. rvm 설치 방법은 Bigmatch님 포스트 [멘붕없이 RVM과 루비 설치하기](http://bigmatch.i-um.net/2013/12/%EB%A9%98%EB%B6%95%EC%97%86%EC%9D%B4-rvm%EA%B3%BC-%EB%A3%A8%EB%B9%84-%EC%84%A4%EC%B9%98%ED%95%98%EA%B8%B0/)를 참고했습니다.

rvm 설치에는 single user와 multi user를 위한 설치법이 있는데, 특별히 서브시스템에 사용자를 추가할 일이 없을 듯 하므로 single user 설치 방법을 따라갑니다.
root 퍼미션이 아닌 채로 아래 명령을 실행합니다.

    $ \curl -L https://get.rvm.io | bash -s stable

설치를 마치면 ~/.bashrc 파일에 다음 줄을 추가합니다. 추가하지 않으면 rvm 명령을 사용할 때 'rvm is not a function' 이라는 오류메세지가 뜹니다.

    # in ~/.bashrc
    source ~/.rvm/scripts/rvm

### ruby 설치
**sudo 붙이지 않고** 아래 명령을 실행합니다. 포스팅 작성 시점에서 ruby latest stable version은 2.4.0입니다.

    $ rvm install [ruby-version]

설치 중간에 시스템 업데이트를 위해 3번정도 sudo 권한을 요구합니다. 물어볼 때 마다 비밀번호를 쳐주면 됩니다.
rvm에는 여러 버전의 ruby가 있을 수 있습니다. list 명령으로 현재 관리되고 있는 ruby 버전을 확인할 수 있고,

    $ rvm list

사용하고자 하는 ruby 버전을 활성화 하려면 use 명령을 사용합니다.

    $ rvm use [installed-ruby-version]

## bundler 설치
ruby의 여러 패키지를 관리하기 위해서 bundler를 설치해주는게 좋습니다.
bundler는 jekyll page에서 필요한 여러 gem을 쉽게 다운로드하고 버전 관리 또한 손 쉽게 해줍니다. 설치하면 삶이 편해집니다.

    $ gem install bundler

## jekyll 설치 및 블로그 생성
이제 jekyll을 설치하고 블로그를 생성합니다. 포스팅 시점에서 jekyll 최신버전은 3.4.0인데 github-page가 사용하는 버전은 3.3.1이었습니다. 그 외 gem의 dependencies는 아래 페이지에서 확인할 수 있는데, jekyll과 필요한 gem을 설치하기 전에 버전을 확인하시는 것 이 좋습니다. [Ggithub page dependency versions](https://pages.github.com/versions/).

    $ gem jekyll install -v 3.3.1
    $ jekyll new YOUR-GITHUB-PAGE-REPO-NAME
    $ cd YOUR-GITHUB-PAGE-REPO-NAME

이제 기본적인 jekyll blog가 생성되었습니다.
잘 생성되었는지 확인하기 위해 아래 명령으로 블로그 서비스를 시작해봅시다.

    $ jekyll serve --no-watch
    Configuration file: /mnt/d/sark/git/alleysark.github.io/_config.yml
    Configuration file: /mnt/d/sark/git/alleysark.github.io/_config.yml
                Source: /mnt/d/sark/git/alleysark.github.io
        Destination: /mnt/d/sark/git/alleysark.github.io/_site
    Incremental build: disabled. Enable with --incremental
        Generating...
                        done in 4.905 seconds.
    Auto-regeneration: disabled. Use --watch to enable.
    Configuration file: /mnt/d/sark/git/alleysark.github.io/_config.yml
        Server address: http://127.0.0.1:4000/
    Server running... press ctrl-c to stop.

정상적으로 서비스가 시작되었다면 http://127.0.0.1:4000/ 에서 페이지가 열릴 것입니다!

> 원래 watch 옵션이 기본으로 켜져 jekyll 페이지에 새 글이 등록되면 jekyll 서비스가 감지하여 포스트를 새로 로딩해줍니다. 그런데 windows bash에선 모니터 서비스가 제대로 동작하지 않는다고 합니다. 이슈가 해결되었다는 포스트를 봤는데.. 저는 안되네요. 추 후 bash 빌드가 올라가면 --no-watch 옵션을 주지 않고도 제대로 서비스가 시작될 수 있습니다.

# 그 외 참고한 글
[Setting up a Jekyll blog using Windows 10 Bash and hosting on GitHub Pages](http://keerats.com/blog/2016/setup-jekyll-blog-windows-10-bash-host-github-pages/)

[Jekyll on Bash on Ubuntu on Windows](http://daverupert.com/2016/04/jekyll-on-windows-with-bash/)