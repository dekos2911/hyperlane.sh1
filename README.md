#!/usr/bin/env bash

channel_logo() {
  echo -e '\033[0;31m'
  echo -e '██████╗ ███████╗██╗  ██╗ ██████╗ ███████╗'
  echo -e '██╔══██╗██╔════╝██║ ██╔╝██╔════╝ ██╔════╝'
  echo -e '██║  ██║█████╗  █████╔╝ ██║  ███╗███████╗'
  echo -e '██║  ██║██╔══╝  ██╔═██╗ ██║   ██║╚════██║'
  echo -e '██████╔╝███████╗██║  ██╗╚██████╔╝███████║'
  echo -e '╚═════╝ ╚══════╝╚═╝  ╚═╝ ╚═════╝ ╚══════╝'
  echo -e '\e[0m'
  echo -e "\n\nПідпишіться на найкращий крипто-канал @indusUA [🚀]"
}

download_node() {
  if [ -f "wallet_hyperlane.json" ]; then
    echo "Нода вже існує. Виконання зупинено. Видаліть ноду, якщо хочете встановити знову"
    exit 1
  fi

  echo 'Починаю встановлення ноди...'

  cd $HOME

  sudo apt-get update -y && sudo apt-get upgrade -y
  sudo apt-get install wget make tar screen nano build-essential unzip lz4 gcc git jq -y

  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash

  export NVM_DIR="$HOME/.nvm"
  [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
  [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

  eval "$(cat ~/.bashrc | tail -n +10)"

  nvm install 20

  if ! command -v docker &> /dev/null; then
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh
    sudo usermod -aG docker $USER
    rm get-docker.sh
  else
    echo "Docker вже встановлено. Пропускаємо"
  fi

  if ! command -v docker-compose &> /dev/null; then
    sudo curl -L "https://github.com/docker/compose/releases/download/$(curl -s https://api.github.com/repos/docker/compose/releases/latest | jq -r .tag_name)/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
  else
    echo "Docker-Compose вже встановлено. Пропускаємо"
  fi

  curl -L https://foundry.paradigm.xyz | bash
  eval "$(cat ~/.bashrc | tail -n +10)"
  foundryup

  cast wallet new --json > wallet_hyperlane.json

  npm install -g @hyperlane-xyz/cli

  docker pull --platform linux/amd64 gcr.io/abacus-labs-dev/hyperlane-agent:agents-v1.0.0

  PRIVATE_KEY=$(jq -r '.[0].private_key' wallet_hyperlane.json)
  ADDRESS=$(jq -r '.[0].address' wallet_hyperlane.json)

  echo -e "Ось ваші дані гаманця: "
  echo -e "Приватний ключ: $PRIVATE_KEY"
  echo -e "Адреса гаманця: $ADDRESS"
  echo -e "Поповніть цю адресу в мережі Base (ETH) на кілька доларів"
}

run_validator() {
  FILE="wallet_hyperlane.json"

  CHAIN="base"
  NODE_NAME=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 8)
  PRIVATE_KEY=$(jq -r '.[0].private_key' wallet_hyperlane.json)
  
  while true; do
    read -p "Введіть ваше RPC посилання для Base Mainnet: " RPC_CHAIN
  
    if [[ $RPC_CHAIN == https://* ]]; then
      echo "Посилання вірне!"
      break
    else
      echo "Помилка: посилання має починатись з https://. Спробуйте ще раз."
    fi
  done

  jq --arg node_name "$NODE_NAME" '
    map(. + {node_name: $node_name})
  ' "$FILE" > "${FILE}.tmp"

  mv "${FILE}.tmp" "$FILE"

  mkdir -p $HOME/hyperlane_db_$CHAIN
  chmod -R 777 $HOME/hyperlane_db_$CHAIN

  docker run -d \
    -it \
    --name hyperlane \
    --mount type=bind,source=$HOME/hyperlane_db_$CHAIN,target=/hyperlane_db_$CHAIN \
    gcr.io/abacus-labs-dev/hyperlane-agent:agents-v1.0.0 \
    ./validator \
    --db /hyperlane_db_$CHAIN \
    --originChainName $CHAIN \
    --reorgPeriod 1 \
    --validator.id $NODE_NAME \
    --checkpointSyncer.type localStorage \
    --checkpointSyncer.folder $CHAIN \
    --checkpointSyncer.path /hyperlane_db_$CHAIN/$CHAIN_checkpoints \
    --validator.key $PRIVATE_KEY \
    --chains.$CHAIN.signer.key $PRIVATE_KEY \
    --chains.$CHAIN.customRpcUrls $RPC_CHAIN
}

check_wallet() {
  PRIVATE_KEY=$(jq -r '.[0].private_key' wallet_hyperlane.json)
  ADDRESS=$(jq -r '.[0].address' wallet_hyperlane.json)

  echo -e "Ось ваші дані гаманця: "
  echo -e "Приватний ключ: $PRIVATE_KEY"
  echo -e "Адреса гаманця: $ADDRESS"
}

check_logs() {
  docker logs -f hyperlane --tail 300
}

restart_node() {
  docker restart hyperlane

  echo 'Нода була перезавантажена.'
}

delete_node() {
  read -p 'Якщо ви впевнені, що хочете видалити ноду, введіть будь-яку букву (CTRL+C щоб вийти): ' checkjust

  echo 'Починаю видалення ноди...'

  cd $HOME

  docker stop hyperlane
  docker rm hyperlane

  sudo rm -r hyperlane_db_base/
  rm wallet_hyperlane.json

  echo "Нода була видалена."
}

exit_from_script() {
  exit 0
}

while true; do
    channel_logo
    sleep 2
    echo -e "\n\nМеню:"
    echo "1. 🔧 Встановити ноду"
    echo "2. ▶️ Запустити валідатора"
    echo "3. 👁️ Переглянути дані гаманця"
    echo "4. 📋 Переглянути логи (вийти CTRL+C)"
    echo "5. 🔄 Перезавантажити ноду"
    echo "6. ❌ Видалити ноду"
    echo -e "7. 🚪 Вийти зі скрипта\n"
    read -p "Оберіть пункт меню: " choice

    case $choice in
      1)
        download_node
        ;;
      2)
        run_validator
        ;;
      3)
        check_wallet
        ;;
      4)
        check_logs
        ;;
      5)
        restart_node
        ;;
      6)
        delete_node
        ;;
      7)
        exit_from_script
        ;;
      *)
        echo "Невірний пункт. Будь ласка, оберіть правильний номер з меню."
        ;;
    esac
  done
