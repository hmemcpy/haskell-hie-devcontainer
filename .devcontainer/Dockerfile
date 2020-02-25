FROM alpine

# && nix-env -i ghc-8.6.5 \

ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID

ARG INSTALL_ZSH="true"
ARG COMMON_SCRIPT_SOURCE="https://raw.githubusercontent.com/microsoft/vscode-dev-containers/master/script-library/common-alpine.sh"
ARG COMMON_SCRIPT_SHA="dev-mode"
RUN apk update \
    && apk add --no-cache wget coreutils ca-certificates \
    && wget -q -O /tmp/common-setup.sh $COMMON_SCRIPT_SOURCE \
    && if [ "$COMMON_SCRIPT_SHA" != "dev-mode" ]; then echo "$COMMON_SCRIPT_SHA /tmp/common-setup.sh" | sha256sum -c - ; fi \
    && /bin/ash /tmp/common-setup.sh "$INSTALL_ZSH" "$USERNAME" "$USER_UID" "$USER_GID" \
    && rm /tmp/common-setup.sh

RUN usermod -a -G wheel ${USERNAME}
RUN echo "%wheel ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

# Enable HTTPS support in wget and set nsswitch.conf to make resolution work within containers
RUN apk add --no-cache --update openssl \
    && echo hosts: dns files > /etc/nsswitch.conf

USER ${USER_UID}:${USER_GID}
WORKDIR /home/${USERNAME}

# Download Nix and install it into the system.
RUN wget https://nixos.org/releases/nix/nix-2.3/nix-2.3-x86_64-linux.tar.xz \
    && tar xf nix-2.3-x86_64-linux.tar.xz \
    && sudo mkdir -m 0755 /etc/nix \
    && echo -e 'sandbox = false\nkeep-derivations = false' > sudo tee /etc/nix/nix.conf \
    && sh nix-*-x86_64-linux/install \
    && sudo rm -r nix-*-x86_64-linux* \
    && sudo rm -rf /var/cache/apk/*

ARG NIXUSR=/nix/var/nix/profiles/per-user/${USERNAME}
ARG NIXPROF=${NIXUSR}/profile

ONBUILD ENV \
    USER=${USERNAME} \
    PATH=${NIXPROF}/bin:${NIXPROF}/sbin:/home/${USERNAME}/.local/bin:/usr/local/bin:/bin:/sbin:/usr/bin:/usr/sbin \
    NIX_PATH=${NIXUSR}/channels

ENV \
    USER=${USERNAME} \
    PATH=${NIXPROF}/bin:${NIXPROF}/sbin:/home/${USERNAME}/.local/bin:/usr/local/bin:/bin:/sbin:/usr/bin:/usr/sbin \
    NIX_PATH=${NIXUSR}/channels

RUN echo ". ${NIXPROF}/etc/profile.d/nix.sh" >> ~/.profile \
    && echo ". ${NIXPROF}/etc/profile.d/nix.sh" >> ~/.zshrc \
    # Make sure the channel is gcrooted, because otherwise it wouldn't be kept with keep-derivations turned off
    && nix-channel --update \
    && nix-collect-garbage --delete-old \
    && nix-store --optimise \
    && nix-store --verify --check-contents

ENTRYPOINT ["/bin/zsh"]

COPY --chown=${USERNAME}:${USERNAME} stack-config.yaml /home/${USERNAME}/.stack/config.yaml
COPY --chown=${USERNAME}:${USERNAME} stack-global-config.yaml /home/${USERNAME}/.stack/global-project/stack.yaml

RUN nix-env -i stack
RUN nix-env -i ghc-8.6.5

RUN nix-env -iA cachix -f https://cachix.org/api/v1/install  \
    && cachix use all-hies \
    && nix-env -iA \
    unstableFallback.selection \
    --arg selector 'p: { inherit (p) ghc865; }' \
    -f "https://github.com/infinisil/all-hies/tarball/85fd0be92443ca60bb649f8e7748f785fe870b7a"

RUN stack install haskell-dap ghci-dap haskell-debug-adapter

RUN nix-collect-garbage --delete-old \
    && nix-store --optimise \
    && nix-store --verify --check-contents
