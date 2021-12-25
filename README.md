# token-updater

[GitHub Actionsでトークンを自動更新する](https://qiita.com/MxShun/items/3c2be23c7e83a8bd2b36) - Qiita

```yml
name: Token Updater

on:
  schedule: # 3か月に1度起動
　   - cron: '* 0 1 */3 *'
  workflow_dispatch: # 手動で起動できるようにしておく

jobs:
  update-the-token:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout the default branch
        uses: actions/checkout@v2
      - name: Set the git configurations
        run: |
          git config user.name ${{ secrets.GITHUB_USER_NAME }}
          git config user.email ${{ secrets.GITHUB_USER_EMAIL }}
      - name: Add a new token
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          TODAY=`date +'%Y-%m-%d'`
          BRANCH_NAME="add/token-${TODAY}"
          echo "ADDITION_BRANCH_NAME=${BRANCH_NAME}" >> $GITHUB_ENV
          git checkout -b $BRANCH_NAME

          TOKEN=$(echo `date` | sha256sum | sed -e "s/.\{3\}$//") # 現在日時 date のハッシュ値を TOKEN とする
          echo "TOKEN=${TOKEN}" >> $GITHUB_ENV
          NEW_PROPERTY=$(cat ./application.properties | sed -e "/^token=[0-9a-zA-Z]\{64\}/s/$/,`echo $TOKEN`/") # 旧トークンの末尾に「,TOKENの値」を追記する
          cat <<< $NEW_PROPERTY > ./application.properties

          git add ./application.properties
          git commit -m "Add a new token on ${TODAY}"
          git push -u origin $BRANCH_NAME
          hub pull-request -b ${{ github.event.repository.default_branch }} -h $BRANCH_NAME -m "API Token定期更新 新規トークンの追加" # デフォルトブランチに対してPRを作成する
      - name: Truncate the old token
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          SWITCH_DATE=`date -d '14 days' +'%Y-%m-%d'`
          echo "SWITCH_DATE=${SWITCH_DATE}" >> $GITHUB_ENV
          BRANCH_NAME="truncate/token-${SWITCH_DATE}"
          git checkout -b $BRANCH_NAME

          NEW_PROPERTY=$(cat ./application.properties | sed -e "s/^token=[0-9a-zA-Z]\{64\},/token=/") # 「token=旧トークンの値」を「token=」に書き換える
          cat <<< $NEW_PROPERTY > ./application.properties

          git add ./application.properties
          git commit -m "Truncate the old token on ${SWITCH_DATE}"
          git push -u origin $BRANCH_NAME
          hub pull-request -b $ADDITION_BRANCH_NAME -h $BRANCH_NAME -m "API Token定期更新 旧トークンの無効化【切替日時：${SWITCH_DATE}】" # 追記するブランチに対して削除するPRを作成する
```