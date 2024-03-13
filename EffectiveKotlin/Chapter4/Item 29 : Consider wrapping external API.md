제작자가 불안정하다고 명시한 경우와 제작자가 안정적으로 유지할 것이라고 믿지 못하는 경우에 불안정한 API를 지나치게 사용하는 것은 불가피하게 API가 변경될 수 있다는 리스크를 가진다.
떄문에, API가 변경될 경우 모든 사용 사례를 수정 해야 한다는 점을 기억하고, 가능한 비지니스 로직 내에서의 사용을 자제하고 분리 시키는 것을 고려해야 한다.

이런 리스크로 '불안정한 external API'는 프로젝트에서 자체적으로 래핑하는 경우가 많으며,   
래핑을 통해 다음과 같은 자율성과 안정성을 확보할 수 있다.

- 래퍼 내부에서의 한 곳만 수정하면 되기에 API 변경에 두려워할 필요가 없다.
- 프로젝트 스타일과 로직에 맞게 API를 조정할 수 있다.
- 라이브러리 이슈가 있는 경우, 쉽게 다른 라이브러리로 교체할 수 있다.
- 필요에 따라 래핑된 객체의 동작을 변경할 수 있다.

하지만, 이런 접근 방식에 대한 반대 의견도 있다.

- 모든 래퍼를 정의 해야 하기에, 'external API'가 업데이트 되면 모든 래퍼도 업데이트 해야 하는 비용이 발생한다. 
- 'internal API'는 해당 프로젝트에만 사용되므로, 이 프로젝트의 개발자들은 또 다른 새로운 API를 배워야 한다.
- 'internal API'에 대한 교육 자료가 없기에, 외부 커뮤니티에 대한 도움을 기대하기 어렵다.

두 의견 모두를 이해한 뒤 래핑할 API를 결정해야 한다.  
라이브러리가 얼마나 안정적인지 판단하는 좋은 지표는 버전 번호와 사용자 수이다.
API의 작은 변경으로 많은 프로젝트에서 수정이 발생한다는 것을 제작자들이 알고 있으면, API 변경에 더욱 신중을 가할 수 밖에 없다.
이 때문에 라이브러리의 사용자가 많을수록 안정적인 API로 판단할 수 있다.
반대로, 가장 위험한 라이브러리는 사용자가 적고 새로운 라이브러리로 판단 할 수 있다. 

이를 토대로, 라이브러리를 현명하게 사용하고 내부적으로 제어할 수 있도록 자체 클래스와 함수로 래핑하는 것을 고려해야 한다.