---
title: test2
date: 2019-07-19 17:39:08
tags:
---

test
test
test
test
test
function getPraiseList(cb){
            axios.post(Config.serverUrl + "/visitAndSign/getpraiseInfo", {
                "signId": Number(self.id),
            }, {
                headers: {'Content-Type': 'application/json', 'DING_TOKEN': self.token}
            }).then(function (res) {
                if (res.data.isSuc) {
                    let list = res.data.result;
                    if(cb){
                        cb(list);
                    }
                }
            })
        }
}
test
test
test
test
test
