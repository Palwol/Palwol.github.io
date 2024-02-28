---
title: react-hook-form으로 재사용 가능한 input 컴포넌트 만들기
date: 2024-02-27 +0900
categories: [react]
tags: [RHF, react]
img_path: /assets/img/posts/2024-02-27
image: react-hook-form-logo.png
render_with_liquid: false
---

[react-hook-form](https://www.react-hook-form.com/)은 React에서 input을 사용할 때 유효성 검사와 각종 상태 관리를 간편하게 할 수 있도록 돕는 라이브러리입니다.
개인적으로 사용하면서 가장 만족하는 React 라이브러리 중 하나인데요.

이 react-hook-form의 `useController` hook을 이용해서 재사용 가능한 input 컴포넌트를 만들어 본 과정을 소개하고자 합니다.

## 1. react-hook-form 사용법

먼저 react-hook-form에서 어떻게 input을 관리하는지 간단히 살펴보겠습니다. react-hook-form에는 `register`를 이용한 비제어 방식과 `Controller`를 이용한 제어 방식 두 가지가 있습니다.

### 1-1. register

먼저 `useForm`의 `register`를 이용하는 방식입니다.  
`register`는 비제어 방식으로 동작합니다.
즉 `input`의 `value`를 `input` 외부에서 변경할 수 없고, 오직 입력값으로만 `value`가 변경됩니다.

> `input`은 기본적으로 **비제어 방식**입니다. `defaultValue`로 초기 `value`값은 설정할 수 있지만, 이후의 `value`는 `input`의 입력을 통해서만 변경할 수 있습니다.  
React에서 `state`를 이용해 `input`의 `value`를 직접 변경하는 방식을
[제어 방식(Controlled)](https://react.dev/reference/react-dom/components/input#controlling-an-input-with-a-state-variable){:target="\_blank"}이라고 합니다.
{: .prompt-info}

```tsx
import { useForm, SubmitHandler } from "react-hook-form";

type TUserInput = {
  firstName: string;
}

const Form = () => {
  const { register, formState: { errors } } = useForm<TUserInput>({
    defaultValues: {
      firstName: ''
    }
  });

  return (
    <form>
      <label>First Name</label>
      <input
        {...register("firstName", {
          required: "이름을 입력해주세요.",
          maxLength: {
            value: 10,
            message: "이름은 10자를 넘을 수 없습니다.",
          },
        })}
      />
      <span>{errors.firstName?.message}</span>
    </form>
  );
};
```

[`register`](https://react-hook-form.com/docs/useform/register)는 input에 자신의 반환값들을 속성으로 넘겨서 사용합니다.
`register`의 첫 번째 인자로 input의 name을 받고, 두 번째 인자로 validation rules를 포함한 옵션 객체를 받습니다.

`register`를 이용하면 손쉽게 비제어 컴포넌트의 validation을 작성할 수 있습니다.
또한 `useForm`의 `formState.errors` 객체가 input에서 발생한 에러 상태를 담고 있어 에러메세지도 간편하게 띄울 수 있습니다.

또한, `register`를 사용하면 input이 비제어 방식으로 동작하기 때문에 입력값이 변경될 때 불필요한 리렌더링이 발생하지 않는다는 장점이 있습니다.
그러나 input 외부에서 value를 직접 지정하는 **제어 방식**을 사용해야 할 때는 `register`를 사용할 수 없습니다.

### 1-2. Controller

input 외부에서 value값 지정이 필요한 제어 컴포넌트를 구현할 때는 `Controller`를 사용합니다.
주로 UI 라이브러리의 input 컴포넌트들이 제어 방식으로 구현되어 있어 `Controller`와 함께 사용하는 경우가 많습니다.

```tsx
import { Input } from "@components/Input";
import { useForm, Controller } from "react-hook-form";

type TUserInput = {
  firstName: string;
}

const Form = () => {
  const { control } = useForm<TUserInput>({
    defaultValues: {
      firstName: ''
    }
  });

  return (
    <form>
      <Controller
        control={control}
        name="firstName"
        rules={{ required: true }}
        render={({ field: { value, onChange, ref }, fieldState: { error } }) => (
          <Input
            value={value}
            onChange={onChange}
            ref={ref} // 에러가 발생한 input에 focus하기 위함
            errorMessage={error?.message}
          />
        )}
      />
    </form>
  );
}
```

[`Controller`](https://react-hook-form.com/docs/usecontroller/controller)는 컴포넌트를 `useForm`에 등록하고 제어하는 Wrapper 컴포넌트입니다.

`Controller`를 사용하려면 `useForm`의 `control`을 props로 넘겨받아야 합니다.
그러면 `render` 함수가 반환하는 컴포넌트가 `useForm`의 제어 컴포넌트로 등록되며, `Controller`가 해당 컴포넌트를 제어할 수 있게 됩니다.

`Controller`는 `field.value`, `field.onChange`, `fieldState.error`와 같이 input의 `value`, `error` 상태를 제어할 수 있는 상태 및 함수를 제공합니다.  
기존에는 React에서 제어 방식 input을 구현하려면 `useState`나 상태관리 라이브러리를 이용해 `value`와 `error` 상태를 따로 관리해줘야 했습니다.
그러나 `Controller`를 사용하면 이와 같은 상태 관리를 `Controller`가 담당하기 때문에 코드가 매우 간결해집니다.

> `field.ref`는 `Controller`에서 input을 제어하기 위해 필수로 넘겨야 하는 값은 아닙니다.
다만 `ref`를 input의 ref로 넘김으로써 input에 validation 오류와 같은 에러가 발생했을 때 해당 input에 focus 되도록 할 수 있습니다.
{: .prompt-tip}

## 2. 재사용 가능한 input 만들기
앞서 살펴본 `register`나 `Controller`를 사용하면 재사용 가능한 input을 만들 수 있습니다.

### 2-1. register 사용 시
컴포넌트에 `regsister`를 적용하려면 `register`의 반환값인 `...register`를 props로 받아서 input 요소에 적용하면 됩니다.
`register`의 반환값인 `UseFormRegisterReturn`의 타입을 살펴보면 다음과 같습니다.

![UseFormRegisterReturn](UseFormRegisterReturn.png)

익숙한 속성들이 많이 보이는데요. `ref`를 제외하면 모두 input 요소 자체에도 존재하는 속성들입니다.
그럼 `register`를 사용하려면 `ref`와 input 요소의 속성들을 넘겨받으면 되겠네요!

```tsx
import { useForm } from "react-hook-form";
import { firstNameRules } from '@static/formValidation';

interface IInputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  errorMessage?: string;
}

const Input = React.forwardRef<HTMLInputElement, IInputProps>(
  ({ label, errorMessage, id, ...props }, ref) => {
    return (
      <Container>
        <label htmlFor={id}>{label}</label>
        <input id={id} ref={ref} {...props} />
        <span>{errorMessage}</span>
      </Container>
    );
  }
);

type TUserInput = {
  firstName: string;
}

const Form = () => {
  const { register, formState: { errors } } = useForm<TUserInput>({
    defaultValues: {
      firstName: ''
    }
  });

  return (
    <form>
      <Input 
        label="이름" 
        errorMessage={errors.firstName?.message} 
        {...register("firstName", firstNameRules)}
      />
    </form>
  );
}
```

`ref`를 props로 넘겨받기 위해서는 [`forwardRef`](https://react.dev/reference/react/forwardRef)로 컴포넌트를 감싸줘야 합니다.
또한 `Input` 컴포넌트가 input 요소의 속성들도 props로 받을 수 있도록 React에서 제공하는 `InputHTMLAttributes`를 상속받아서 `IInputProps`를 구현했습니다.

### 2-2. Controller 사용 시
input 컴포넌트를 제어 방식으로 사용하려면 `register` 대신 `Controller`를 사용해야 합니다. 
`Controller` 사용 시에도 재사용 가능한 `Input`을 구현하는 방법은 `register`와 동일합니다. 

```tsx
import { useForm, Controller } from 'react-hook-form';
import { firstNameRules } from '@static/formValidation';

// register와 동일
interface IInputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  errorMessage?: string;
}

const Input = React.forwardRef<HTMLInputElement, IInputProps>(
  ({ label, errorMessage, id, ...props }, ref) => {
    return (
      <Container>
        <label htmlFor={id}>{label}</label>
        <input id={id} ref={ref} {...props} />
        <span>{errorMessage}</span>
      </Container>
    );
  }
);

type TUserInput = {
  firstName: string;
}

// Controller 사용
const Form = () => {
  const { control } = useForm<TUserInput>({
    defaultValues: {
      firstName: ''
    }
  });

  return (
    <form>
      <Controller 
        name="firstName"
        control={control}
        rules={firstNameRules}
        render={({ field: { value, onChange, ref }, fieldState: { error } }) => (
          <Input
            value={value}
            onChange={onChange}
            ref={ref} // focus가 필요한 경우 사용
            label="이름"
            errorMessage={error?.message}
          />
        )}
      />
    </form>
  );
}
```

이렇게만 해도 충분히 재사용성이 높은 input 컴포넌트를 구현할 수 있지만, 여전히 `register`를 넘기거나 `Controller`를 사용해야 하는 수고가 들어갑니다.
또한 컴포넌트 내부에서 `error`까지 관리할 수 있으면 더욱 좋을 것 같다는 생각이 드는데요.

이를 모두 해결해주는 구원자가 바로 `useController`입니다.

### 2-3. useController 이용하기

`useController`는 `Controller`를 동작시키는 custom hook입니다. `Controller`와 동일하게 제어 방식으로 input을 컨트롤하나,
컴포넌트 외부에서 `render` 함수로 컴포넌트를 전달받는 `Controller`와 달리 컴포넌트 내부에서 `useController` hook 호출을 통해 컴포넌트를 제어하기 때문에
재사용성이 더 높은 제어 컴포넌트 구현이 가능합니다.

```tsx
import { useForm, useController, UseControllerProps } from "react-hook-form";
import { firstNameRules } from '@static/formValidation';

type TUserInput = {
  firstName: string;
};

interface IInputProps {
  label: string;
  id: string;
}

const Input = ({
  label,
  id,
  ...props
  }: IInputProps & UseControllerProps<TUserInput>) => {
  const {
    field: { value, onChange },
    fieldState: { error }
  } = useController(props); // 컴포넌트 내부에서 hook 호출

  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} value={value} onChange={onChange} />
      <span>{error?.message}</span>
    </div>
  );
}

 const Form = () => {
  const { control } = useForm<TUserInput>({
    defaultValues: {
      firstName: ''
    }
  });

  return (
    <form>
      <Input name="firstName" control={control} rules={firstNameRules} />
    </form>
  );
}
```
`Input` 컴포넌트를 사용하는 코드가 놀랍도록 간단해진게 보이시나요?
기존에 `Controller`의 `render`에서 주입하던 `value`, `onChange`, `error`와 같은 속성을 컴포넌트 내부에서 사용할 수 있어
컴포넌트 외부에서는 `UseControllerProps`에 해당하는 props를 받아서 넘기기만 하면 끝입니다.

`UseControllerProps`의 타입은 다음과 같습니다.

![UseControllerProps](UseControllerProps.png)

위 타입에 의하면 name만 필수 요소이고 나머지는 선택 요소입니다. 그러나 실제 동작 시에는 `control`을 넘기지 않으면 `useController` hook에서 에러가 발생합니다.
[공식문서](https://www.react-hook-form.com/api/usecontroller/)에 의하면 `control`은 기본적으로 필수 요소이나,
`useController`를 사용한 컴포넌트가 [`FormProvider`](https://www.react-hook-form.com/api/formprovider/)로 감싸져 있다면 `control`을 받지 않아도 된다고 합니다.
(`FormProvider`의 context에서 `control`이 제공되는 듯 합니다.)

<br />
이제 정말 재사용성이 높은 컴포넌트가 만들어진 것 같았지만..! 치명적인 오류가 하나 있습니다.
바로 `UseControllerProps`가 제네릭으로 `useForm`의 input name으로 사용될 타입(`TUserInput`)을 받고 있다는 점입니다.

```tsx
...

type TUserInput = {
  firstName: string;
};

const Input = ({
    ...
  }: IInputProps & UseControllerProps<TUserInput>) => { // 이 부분..
    ...
  }
```

이렇게 되면 `Input` 컴포넌트를 만들 때 미리 input의 name으로 사용할 타입을 지정해줘야 하기 때문에 재사용 가능한 컴포넌트를 구현할 수 없습니다.
이를 해결하기 위해서는 `Input` 컴포넌트를 [제네릭 컴포넌트](https://ui.toast.com/posts/ko_20210505)로 만들어줘야 합니다.

제네릭 컴포넌트 작성을 위해 `UseControllerProps` 타입을 다시 확인해봅시다.

![UseControllerProps with generic](UseControllerProps-generic.png)

`UseControllerProps`는 `TFieldValues`와 `TName` 두 가지 인자를 제네릭으로 받아서 사용합니다.
이 두 인자를 `Input`의 제네릭 인자로 받아서 `UseControllerProps`에 제공해주면 되겠네요!

`TFieldValues`는 `control`, `TName`은 `name`의 타입으로 사용되기 때문에 이 두 props에서 제네릭 타입 추론이 가능합니다.

```tsx
import { 
  FieldPath, 
  FieldValues, 
  useController, 
  UseControllerProps, 
  useForm 
} from 'react-hook-form';

interface IInputProps {
  label: string;
  id: string;
}

const Input = <
  TFieldValues extends FieldValues = FieldValues,
  TName extends FieldPath<TFieldValues> = FieldPath<TFieldValues>
>({
  label,
  id,
  ...props
}: IInputProps & UseControllerProps<TFieldValues, TName>) => {
  const {
    field: { value, onChange },
    fieldState: { error }
  } = useController(props);

  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} value={value} onChange={onChange} />
      <span>{error?.message}</span>
    </div>
  );
}

type TUserInput = {
  firstName: string;
};

const Form = () => {
  const { control } = useForm<TUserInput>({
    defaultValues: {
      firstName: ''
    }
  });

  return (
    <form>
      /* name, control에서 TName, TFieldValues 타입 추론 */
      <Input name="firstName" control={control} rules={firstNameRules} />
    </form>
  );
}
```

이로써 드디어 완전히 재사용 가능한 input 컴포넌트를 만들었습니다!

## 마치며

이렇게 작성한 컴포넌트를 매우 만족스럽게 사용하고 있지만, 여전히 약간 아쉬운 점이 있습니다.
`useController`를 사용한 컴포넌트에는 반드시 `control`을 넘기거나 `FormProvider`로 컴포넌트를 감싸야 하기 때문에,
UI만 필요한 경우 등 react-hook-form과 함께 사용하지 않을 때는 컴포넌트 재사용이 어렵다는 점입니다.  
이를 해결하기 위해 UI만 있는 컴포넌트를 따로 구현하거나 `useController`가 적용된 wrapper 컴포넌트를 구현하는 등 여러 시도를 해봤지만
아직 최선의 해결책을 찾지는 못한 것 같습니다. 이 부분은 조금 더 고민이 필요할 것 같네요.

react-hook-form을 확장성 있게 사용하시고 싶은 분들에게 이 글이 조금이나마 도움이 되기를 바랍니다.  
의견이나 오류 제보는 댓글 부탁드립니다!  
감사합니다.