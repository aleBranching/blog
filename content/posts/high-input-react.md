---
date: '2025-07-06T21:31:59+01:00'
draft: false
title: 'Hight tab, high input react app: Or how I learned to stop listening to react docs and learn data flow.'
cover:
    image: high-input-react/Strangelove.webp
ShowToc: true

---

# Web app design

Firstly ignoring the technical details, let's go over the end goal of the web app. 

![alt text](/blog/high-input-react/image-1.png)
web app design


![alt text](/blog/high-input-react/image-2.png)

We would like to receive data from an API endpoint into our web app. Then within the web app we need to do CRUD (create, read, update, delete) and re-order some of the data before sending it off to the destination.

user flow:

- New tabs can be created for a specific api endpoint. 
- This brings up specific API endpoint parameters to fetch the data. 
- Pressing get data will fetch the data into the data manipulation section. 
- Switching from one tab to another would preserve any data manipulation that happened as well as the API parameters that were selected. 


# Initial thoughts on state


Ignoring React specifics, the natural way to represent the state would be a list of tabs. Each tab having a large JSON needed to render endpoint parameters and the data manipulation. 

Some of the endpoints contain a lot of data needing up to a total of 100 inputs. 

![alt text](/blog/high-input-react/image-4.png)

# React state

The Initiall approach to the project was to use react context. I would not run into prop drilling issue and I wrote a nice abstraction that let me manipulate any data of the current tab. 
This is the reducer function:
```javascript
let reducerFunc = (draft, action) => {
  switch (action.type) {
    case "currentTab":
      const { callback } = action;
      // debugger;
      let theTab = draft.find((ele) => ele.current == true);
      // debugger;
      callback(theTab);

      break;
  }
  }

```
it is created with immer to allow mutating changes:
```javascript
const [tabs, dispatch] = useImmerReducer(reducerFunc, inititialValues);
```


Then in order to change the state of the current tab I can just use a callback function:

```javascript
dispatcher({
  type: "currentTab",
  callback: (currentTabDraft) => {
    delete currentTabDraft.season;
  },
});
```

## Problems of this approach
Any state changes means everything re-renders. Using chrome dev tools, I did CPU slow down to 4x because my computer does not represent the users typical machine. Typing in an input can cause 100ms re-render. This is definitely noticable. While the react dev tools will cause the dev page to be slower, this is still a concerning performance if this project is to be used long term and new features are to be added. 



State still needs to be kept at the top since each tab holds all the information. An unlimited amount of tabs can be created. 


# Rethinking the data flow in React

I need all the data in the data manipulator for 2 reasons: 
- Clicking into that tab 
- sending data button

Going through the react documentation it would lead you to believe that you need to move the state up into any component that needs the data including the button. But I only need that data later on. I don't need to "React" 🤯 to the data changes in my UI (except of course the input itself). 

One day I was browsing MDN documentation of all the web API's when indexdDB jumped into my eye and came up with the following solution:

![alt text](/blog/high-input-react/image-5.png)

I store each tab as an entry to indexdb. Each of these entries is an object with all the data needed to represent the data manipulation component and the endpoint selection (endpoint selection was not drawn for simplicity). 

Clicking into a tab fetches the tab from indexDB. Then you load that data into the data manipulation component. From then on you can play with the state however you want within data manipulation and it will cause only a re-render within the component. You can of course seperate the state into further sub-components to only cause a re-render within that sub-component. The only rule is that any state change needs to be written into indexdDB. Firstly, the state is changed on key-stroke because the ui update is priority and only then written to indexdDB. If the write fails, revert back to old state. 

I won't go over the details of implementing indexDB in react as it has been mentioned elsewhere. I'm trying to keep this post short. 


Here is an example of a subcomponent broken down to just an input:

```typescript
type ASinglePlayerProps = {
  index: number;
  db: IDBDatabase;
  currentTabId: number;
  name: string;
};
let AValue = ({ index, db, currentTabId, passedName }: AValueProps) => {
  const [name, setName] = useState(passedName);

  const handleChange = async (
    e: React.ChangeEvent<HTMLInputElement>
  ) => {
    let newValue = e.target.value;
    let oldValue = name;

    setName(newValue);
    try {
      await updateDBTab<TypeRepresentingTab>(db, currentTabId, (data) => {
        data.previewData.someList[index].name = newValue;
        return data;
      });
    } catch (error) {
      console.error("There was an error updating", error);
      toast.error(
        "Fatal error writting to database\nChanges are not being recorded"
      );
      // in case write fails revert back
      setName(oldValue);
    }
  };

  return (
    <div>
      <input
        type="text"
        onChange={(e) => handleChange(e)}
        value={name}
      />
    </div>
  );
};
export default AValue;
```

On input change only this component will re-render. And now react profiler is showing me 2.7ms re-render with 4x cpu slowdown. This is quite the jump. The application is noticably snappier. 

Then to use the send data button, you just access the database. The component can be placed anywhere within the component tree. 

Additionally indexdb persists data on closing the browser so the same data can be loaded in. 


# Conclusion
This is the solution I came up with and decided to share it as I could not find anything on the internet on how to handle information dense UI's in react.

Of course there may be a state library that already does something similar. These days I'm a bit more apprehensive to add dependencies to my project. I try to keep towards core web technologies. There's nothing worse than coming back to a project and some library like formik is deprecated or there has been breaking changes (shadcn). 

I would also argue that implementing indexdb into react is not that hard. It is also easy to understand what is actually happening compared to a state management library. 
