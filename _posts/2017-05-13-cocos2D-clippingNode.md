---
layout: post
title:  "【Cocos2D + Lua】ClippingNode制作动画"
date:   2017-05-13
excerpt: "今天熟悉了Lua的语法，同时学习了Cocos中`ClippingNode`的使用，这个节点通常是用来做动画的，示例如下："

tag:
- Cocos2D
comments: true
---

>今天熟悉了Lua的语法，同时学习了Cocos中`ClippingNode`的使用，这个节点通常是用来做动画的，示例如下：


![clip.gif](http://ocigwe4cv.bkt.clouddn.com/clip.gif)

### 开始
下面就让我们来利用`ClippingNode`来作出一个类似上面的这种效果！
代码参考如下所示：
```
function init_element()
    self._layer = cc.LayerColor:create(cc.c3b(0,255,0))
        local scale = uiex.ui_fit:min_scale()
	    self._layer:setScale(scale)
	        self._layer:setAnchorPoint(cc.p(0,0))
		    self:addChild(self._layer)

		        local stencil = display.newSprite("image/login/earth_mask.png")
			    local clip = cc.ClippingNode:create(stencil)
			        clip:move(display.cx, display.cy)
				    clip:setAlphaThreshold(0.1)

				        local logo = display.newSprite("HelloWorld.png")
					    logo:setAnchorPoint(0,0)
					        local logoSize = logo:getContentSize()


						    logo:runAction( cc.RepeatForever:create(
						            cc.Sequence:create(
							                cc.MoveBy:create(5, cc.p(-logoSize.width*2,0)),
									            cc.CallFunc:create(
										                    function(  )
												                        logo:setPositionX(0)
															                end)
																	            )))

																		        clip:addChild(logo)
																			end
																			```

																			### 接着
																			其实这个动画的原理如下：
																			>想象有一个洞，然后有一块内容不断的移动同时经过这个洞，这样也就有了上述的动画效果。

																			上述代码中有两个主要的Node，一个是我们利用`stencil`创建的`ClippingNode`,这个就是一个洞；然后我们创建了`logo`,这就是内容了；下面对`logo`进行`runAction`,也就是相当于不断的移动内容；最后需要将内容作为子节点加入到洞中，这样动画就完成了。

																			### 尾巴
																			Lua写Cocos真心蛋疼，尤其是我这样的初学者😂


