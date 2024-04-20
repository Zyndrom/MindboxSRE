# Процес посика и принятия решений.

> у нас мультизональный кластер (три зоны), в котором пять нод

Ищу в интернете, что такое 'мультизональный кластер' и чем он отличается от обычного. 
Нашёл то что это кластеры, разделённые на несколько дата центров.
После чего решил выяснить, как именно это влияет на задание.
Нашёл следующий код.
```
	affinity:
		podAntiAffinity:
			requiredDuringSchedulingIgnoredDuringExecution:
			  labelSelector:
					matchExpressions:
					  key: app
						operator: In
						values:
						  ournoce
				topologyKey: topology.kubernetes.io/zone
```

Решил узнать по подробнее. Нашёл это видео https://www.youtube.com/watch?v=nBWRIlnszmY в котором автор объясняет, что это не лучшее решение. И предлагает следующее:
```
	topologySpreadConstraints:
		  maxSkew: 1
			topologyKey: topology.kubernetes.io/zone
			whenUnsatisfiable: ScheduleAnyway
			labelSelector:
				matchLabels:
					app: ourapp
```
> приложение требует около 5-10 секунд для инициализации
Добавляем readinessProbe в Deployment.
```
	readinessProbe:
		httpGet:
			path: /
			port: 8080
		initialDelaySeconds: 5
		periodSeconds: 5
```

> по результатам нагрузочного теста известно, что 4 пода справляются с пиковой нагрузкой
Сначала решил, что запуск 4 подов сразу будет оптимально и по ночам, исходя из следующего условия, снижать это количество до 1-2 при помощи Cronjob. Но решил по гуглить, как принято справляются с этим. В результате нашёл Horizontalpodautoscaler, который в зависимости от нагрузки запускает нужное количество подов.

> на первые запросы приложению требуется значительно больше ресурсов CPU, в дальнейшем потребление ровное в районе 0.1 CPU. По памяти всегда "ровно" в районе 128M memory
Следственно, ставим requests на Memory 128M, CPU на 150m. А limits с небольшим запасом для нормального запуска приложения.
```
	resources:
		requests:
			memory: 128M
			cpu: 150m
		limits:
			memory: 256M
			cpu: 1
```

> приложение имеет дневной цикл по нагрузке – ночью запросов на порядки меньше, пик – днём
Так как мною было решено использовать HorizontalPodAutoscaler, он сам будет до запускать контейнеры при повышении нагрузки днём.