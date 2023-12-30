FROM python:3.12-bookworm

RUN apt-get update

# RUN apt-get install --no-install-recommends -y git procps

# Install Tex Live
RUN apt-get update && apt-get -y upgrade \
    && apt-get -y install --no-install-recommends \
    texlive-latex-base \
    texlive-extra-utils \
    texlive-latex-extra \
    biber chktex latexmk
    # make python3-pygments python3-pkg-resources \
    # texlive-lang-chinese \
    # texlive-lang-japanese

# latexindent modules
# RUN apt-get install --no-install-recommends -y curl

RUN curl -L http://cpanmin.us | perl - App::cpanminus \
    && cpanm Log::Dispatch::File \
    && cpanm YAML::Tiny \
    && cpanm File::HomeDir \
    && cpanm Unicode::GCString

# Clean up
RUN apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*
ENV DEBIAN_FRONTEND=dialog \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8

RUN pip install --upgrade pip

COPY ../reqs/requirements.txt reqs/

RUN pip install -r reqs/requirements.txt