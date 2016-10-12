# Операция «Плавные машинки»
### или как обмануть юзера, так чтобы он ничего не заметил

Прошел год как мы выпустили новое клиентское приложение. Как-то раз, в один ничем не примечательный день пришел к нам (не помню кто) и говорит: «Как ~~аху~~круто было бы, если в нашем приложении клиент видел как к нему едет его водитель». На то время у нас было:
* Водительское приложение передававшее местоположение раз в 15 секунд;
* Бекэнд, который выдавал последние полученные координаты водителя;
* Маркер в клиентском приложении обозначающий местоположение водителя.

Через некоторое время, разгрузив текущие задачи, мы сделали первый прототип плавных машинок. Машинка начала ездить, но не всегда по дороге. Она могла срезать угол, проехать по зданию или полю, а в некоторых случаях могла проехать пол города. Это происходило из-за большого интервала между полученными координатами. У нас было два варианта исправления этой проблемы:

1. Переделать водительское приложение, так чтобы оно отсылало полученные координаты сразу при их изменении;
2. Вернуть в клиентское приложение маршрут, по которому водитель мог проехать от последней сохраненной координаты до текущего местоположения.

Возможности (как и желания), на тот момент, переделать водительское приложение у нас не было, так как это значительно увеличило бы нагрузку на сервер и расходы на трафик у водителя. К тому же отсутсвие качественных GPS модулей на мобильных устройствах наших водителей мешало получению точных координат, и как следствие маркеры все равно продолжили бы ездить по зданиям.

Остановившись на втором варианте мы сделали следующее:

1. Водительское приложение не изменилось;
2. На бекэнде был добавлен метод, который возвращает маршрут от полученной координаты до текущего местоположения водителя;
3. В клиентском приложении первое получение координат водителя не изменилось, а далее мы анимируем движение машинки по полученному маршруту;
4. Для того чтобы компенсировать время на запрос нового маршрута, общее время анимации машинки по маршруту было сделано больше чем интервал между получением маршрутов;
5. Для того чтобы машинка двигалась с одной скоростью на протяжении всего маршрута, время анимации для каждого отрезка вычисляется в зависимости от отношения длинны этого отрезка к общей длинне маршрута.

В итоге в Android приложении получился следующий класс:
```java
public class MarkerAnimator implements Runnable {
    MarkerView marker;

    LatLng position, startPosition;
    float rotation, startRotation;
    int duration, routeAnimationDuration, angle_duration, location_duration;
    boolean routeAnimationIsRunning;
    long startTime;

    ArrayList<LatLng> routePoints;
    float routeLength;

    public MarkerAnimator(MarkerView marker) {
        this.marker = marker;
        position = marker.getPosition();
        routePoints = new ArrayList<>();
        routeAnimationIsRunning = false;
    }

    public LatLng getPosition() {
        if (routePoints.size() > 0) {
            return routePoints.get(routePoints.size() - 1);
        } else {
            return position;
        }
    }

    public boolean markerIsVisible() {
        return marker.isVisible() && marker.getAlpha() == 1.0;
    }

    public void removeMarkerFromMap() {
        marker.remove();
    }

    public void updateMarkerByRoute(ArrayList<LatLng> route, int duration) {
        routePoints.addAll(route);
        routeAnimationDuration = duration;

        //Вычисляем общую длину маршрута
        routeLength = 0;
        for (int i = 1; i < route.size(); i++) {
            routeLength += route.get(i - 1).distanceTo(route.get(i));
        }

        if (!routeAnimationIsRunning) {
            routeAnimationIsRunning = true;
            new Runnable() {
                @Override
                public void run() {
                    if (routePoints.size() > 0) {
                        double distance = marker.getPosition().distanceTo(routePoints.get(0));

                        //Рассчитываем время анимации машинки до следующей точки
                        int duration = 0;
                        if (distance > 0) {
                            duration = routeAnimationDuration / routePoints.size();
                            if (routeLength > 0 && routeLength >= distance) {
                                duration = (int) ((distance / routeLength) * routeAnimationDuration);
                            }

                            //Запускаем анимацию машинки до следующей точки
                            updateMarker(routePoints.get(0), duration);

                            routeAnimationDuration -= duration;
                        }
                        //Перезапускаем Runnable, чтобы анимировать движение машинки до следующей точки
                        new Handler().postDelayed(this, duration);
                        routePoints.remove(0);
                    } else {
                        routeAnimationIsRunning = false;
                    }
                }
            }.run();
        }
    }

    //Функция расчета угла между текущим положением машинки и новыми координатами
    public void updateMarker(LatLng position, int duration) {
        float rotation;
        double lat1 = this.position.getLatitude();
        double lng1 = this.position.getLongitude();
        double lat2 = position.getLatitude();
        double lng2 = position.getLongitude();

        if (lat1 == lat2 && lng1 == lng2 && marker != null) {
            rotation = marker.getRotation();
        } else {
            double y = lng2 - lng1;
            double x = lat2 - lat1;
            rotation = (float)(Math.atan2(y, x) * 180 / Math.PI);
        }

        this.position = position;
        this.rotation = rotation;
        this.duration = duration - (duration % 16);

        startPosition = marker.getPosition();
        startRotation = marker.getRotation();
        startTime = SystemClock.uptimeMillis();

        //Исправляем текущий угол для правильной анимации поворота машинки
        while (Math.abs(startRotation - rotation) > 180) {
            if (startRotation - rotation > 180) {
                startRotation -= 360;
            } else if (startRotation - rotation < -180) {
                startRotation += 360;
            }
        }

        angle_duration = this.duration / 6;
        if (angle_duration > 500 && this.duration > 3000) angle_duration = 500;

        location_duration = Math.abs(startRotation - rotation) % 360 < 30 ? this.duration : this.duration - angle_duration;

        run();
    }

    //Функция анимации движения и поворота машинки
    @Override
    public void run() {
        float elapsed = SystemClock.uptimeMillis() - startTime;
        float t_location = (elapsed - (duration - location_duration)) / location_duration;
        float t_angle = elapsed / angle_duration;

        if (t_location < 0) t_location = 0;
        if (t_angle < 0) t_angle = 0;
        if (t_location > 1) t_location = 1;
        if (t_angle > 1) t_angle = 1;

        double lat = startPosition.getLatitude() + (position.getLatitude() - startPosition.getLatitude()) * t_location;
        double lng = startPosition.getLongitude() + (position.getLongitude() - startPosition.getLongitude()) * t_location;

        float angle = t_angle * rotation + (1 - t_angle) * startRotation;

        if (marker != null && !Float.isNaN(angle)) {
            LatLng position = new LatLng(lat, lng);
            if (t_location > 0) marker.setPosition(position);
            marker.setRotation(angle, 0);

            if (t_location < 1) {
                //Перезапускаем данную функцию через 16мс (60fps), пока машинка не доедет до нужной координаты
                new Handler().postDelayed(this, 16);
            }
        }
    }
}
```

В iOS была сделана следующая функция:
```swift
     func moveDriverMarker(_ interval: Double) {
        if (!self.myDriverIterationCoordinate.isEmpty && self.myDriverMarker.map != nil) {
            тут мы запоминаем текущее положение маркера, и запоминаем его rotation

            if (self.myDriverMarker.position.latitude != myDriverIterationCoordinate.first!.latitude && self.myDriverMarker.position.longitude != myDriverIterationCoordinate.first!.longitude) {

                let dY = myDriverIterationCoordinate.first!.longitude - self.myDriverMarker.position.longitude

                let dX = myDriverIterationCoordinate.first!.latitude - self.myDriverMarker.position.latitude

                angle = atan2(dY, dX) * 180 / M_PI

Вычисление нового угла маркера в соотношение текущей его позиции и следующей координате по которой он начнет движение

            }

            CATransaction.begin()

            CATransaction.setAnimationDuration(0.3)

            self.myDriverMarker.rotation = angle

задаем угол поворота маркеру

            CATransaction.commit()

            CATransaction.begin()

            CATransaction.setCompletionBlock({

                if (!self.myDriverIterationCoordinate.isEmpty) {

                    self.myDriverIterationCoordinate.removeFirst()

Удаляем координату по которой у нас началось движение маркера

                    if (Double(self.myDriverIterationCoordinate.count) != 0.0) {

                        self.moveDriverMarker((22.0 - interval) / Double(self.myDriverIterationCoordinate.count))

Вызываем нашу функцию отрисовки заново

                    } else {

Если Координаты концились то мы ждем новых координат от сервара

                    }

                }

            })


            CATransaction.setAnimationDuration(interval)

Задаем интервал анимации маркера

            self.myDriverMarker.position = self.myDriverIterationCoordinate.first!

Задаем позицию маркера

            CATransaction.commit()

        } else {

Если Координаты концились то мы ждем новых координат от сервара

        }

    }
```
